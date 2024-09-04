<svelte:options tag="fds-image-editor-text"></svelte:options>
<script>
    import { onMount } from "svelte"
    
   
    import {get_current_component} from "svelte/internal"
    let host = get_current_component()
    import {		afterUpdate } from 'svelte';
    // eslint-disable-next-line no-unused-vars
    import Dialog from './Dialog.svelte';
        /**
     * width of canvas
     * @type {integer}
     */
    export let width=500
    /**
     * height of canvas
     * @type {integer}
     */    
    export let height=500
    /**
     * object with data property to save scene on destroy 
     */
    export let external_scene_storage=null

    export function refresh() {
        layer=layer
    }
    let timer;
    const debounceDelay = 1000; // 1 second for clipping mask refresh

    afterUpdate(() => {
        if (timer) {
            clearTimeout(timer) // Cancel any previous timeout
        }

        // Start a new timeout to delay image generation by 1 second
        timer = setTimeout(() => {
            generateClippingMasks()
        }, debounceDelay)
    })

    function generateClippingMasks() {
        if (!layer.id) return
        let list = globalThis.gyre.layerManager.findClippingMasks(null, layer.id)
        if (list.length) {  // these are usually only one so optimization is not needed here
            for (let i = 0; i < list.length; i++) {
                list[i].clippingMask(layer) 
            }
        }
    }



    function _destroy() {
        if (!external_scene_storage) return
        external_scene_storage.data=getScene()
    }
    onMount(() => {
        return () => {
            _destroy()
        }
    })    
    

    export async function prepareForAI(resultLayer) {
        let imageAPI=globalThis.gyre.imageAPI
        let canvas=globalThis.gyre.canvas
        width = resultLayer.width
        height = resultLayer.height
        layer.url = await renderForAI()     // set image data
        width = resultLayer._width
        height = resultLayer._height
        // after generated image of text layer croü to selected region
        let clippedCanvas = window.document.createElement('canvas')
        clippedCanvas.width = resultLayer.width
        clippedCanvas.height = resultLayer.height
        let clippedCtx = clippedCanvas.getContext('2d')
        clippedCtx.drawImage(await imageAPI.base64ToCanvas(layer.url), resultLayer.x, resultLayer.y, resultLayer.width, resultLayer.height, 0, 0, resultLayer.width, resultLayer.height);
        let base64=clippedCanvas.toDataURL()
        // scale 
        let scaledBase64 = await imageAPI.scaleImageCanvas(base64, resultLayer.generate.targetWidth, resultLayer.generate.targetHeight)
        let ScaledImageInfo={}
        // fill transparent areas with white
        base64 = await  imageAPI.fillImageBackground(scaledBase64, 'white')
        let HTMLCanvas=await imageAPI.base64ToCanvas(base64)
        // white on black needed for ControlNet
        imageAPI.canvasInvert(HTMLCanvas)
        ScaledImageInfo.base64=ScaledImageInfo.urlRef=HTMLCanvas.toDataURL()
        ScaledImageInfo.type = 'skribble' // use ControlNet skribble
        ScaledImageInfo.base64 = ScaledImageInfo.base64.split(',')[1] // no url prefix for base64 needed here
        resultLayer.specialLayersImageInfo.push(ScaledImageInfo)
        layer.url=null  // temp render image not needed anymore
    }


    export function layerInstance() {
        class textLayer extends globalThis.gyre.getLayerBaseClass() {
            font_size
            string
            align
            bold
            italic
            underline
            letter_spacing
            font
            async renderInDocumentSize() {
                return await renderForAI()
            }
        }
        let newlayer=new textLayer()
        newlayer.type = 'fds-image-editor-text'
        newlayer.name = 'Text'
        newlayer.letter = "T"
        newlayer.background_type = 'transparent'
        newlayer.font_size = 200
        newlayer.string = 'TEXT'
        newlayer.align="middle"
        newlayer.bold=false
        newlayer.italic=false
        newlayer.underline=false
        newlayer.letter_spacing = 0
        newlayer.font = 'Josefin Sans'
        layer=newlayer
        return newlayer
    }

    export async function prepareForSave() {
        layer.data = getScene()
        return layer
    }

    export function getLayerMenuHTML(l,thumbclass,thumbStyle) {
        // justr render a T as thumb. Could be an image as well
        let html=
            '<div ' +
            thumbclass +
            ' style="' +
            thumbStyle +
            ' font-size:33px;font-weight:bold;display:flex;align-content:space-between;justify-content:space-around;align-items: center;">T</div> ' +
            l.name
        return html
    }

    export async function quickMask() {
        let imageAPI=globalThis.gyre.imageAPI
        let canvas=globalThis.gyre.canvas      // this is app main canvas and not an HTML canvas
        width = layer.width
        height = layer.height
        layer.url = await renderForAI()
        width = layer._width
        height = layer._height
        let onelayer = [layer] // only one layer
        let base64 = await imageAPI.getImageDataCombined(onelayer, 0, 0, canvas.width, canvas.height, {width:canvas.width,height:canvas.height}, null, true)
        let HTMLCanvas = await imageAPI.base64ToCanvas(base64, canvas.width, canvas.height) // can be HTML canvas or Tiled canvas
        imageAPI.canvasMakeTransparentWhite(HTMLCanvas)
        imageAPI.canvasInvert(HTMLCanvas)
        HTMLCanvas = await imageAPI.canvasExpandMaskFast(HTMLCanvas)
        let res = await HTMLCanvas.toDataURL()
        return res
    }
    /**
     * check all styleSheets
     * external link or imported stylesheets
     * need to be fetched and inlined
     * to create font Files object
     */
    async function getFontReferences() {
    let fontFiles = [];

    // inline external css
    let styleSheets = await inlineExternalCss(true, true);

    styleSheets.forEach((styleSheet, s) => {
        let cssRules = styleSheet.cssRules;
        for (let i = 0; i < cssRules.length; i++) {
        let rule = cssRules[i];
        let typeRule = rule.type;

        // type===5: is @font-face rule
        if (typeRule === 5) {
            let style = rule.style;
            let fontFamily=style.getPropertyValue("font-family")
            fontFamily = fontFamily.replaceAll('"', "");
            let fontWeight=style.getPropertyValue("font-weight")
            fontWeight = parseFloat(style.fontWeight);
            let fontStyle=style.getPropertyValue("font-style")
            fontStyle = style.fontStyle;
            let unicodeRange = style.unicodeRange;

            let src = style.getPropertyValue("src")
            .split(",")
            .map((val) => {
                return val.split(" ")[0];
            })
            .filter(Boolean);
            let url = "";

            // extract font file URLs/paths
            src.forEach((fontSrc) => {
            url = fontSrc.match(/\(([^)]+)\)/)[1].replaceAll('"', "");
            });

            // add to fontFiles object
            fontFiles.push({
            origin: origins[s],
            fontFamily: fontFamily,
            fontWeight: fontWeight,
            fontStyle: fontStyle,
            src: url,
            unicodeRange: unicodeRange
            });
        }
        }
    });

    return fontFiles;
    }

    /**
     * inline all external stylesheets
     */

    // collect all css origins
    let origins = [];

    async function inlineExternalCss(removeExternalCss = true) {
    let styleSheets = Array.from(document.styleSheets);

    /**
     * stylesheet helper
     * fetch external css
     */
    const fetchExternalStylesheet = async (href, ) => {
        let fetched = await fetch(href);
        let css = await fetched.text();
        let newStyle = document.createElement("style");
        newStyle.classList.add("externalCSS");
        let externalCss = document.head.querySelectorAll(".externalCSS");
        if (!externalCss.length) {
        document.head.insertBefore(newStyle, document.head.children[0]);
        } else {
        document.head.insertBefore(
            newStyle,
            externalCss[externalCss.length - 1].nextElementSibling
        );
        }
        newStyle.textContent = `/** origin: ${href} **/\n` + css;
        origins.push(href);
    };

    for (let i = 0; i < styleSheets.length; i++) {
        let stylesheet = styleSheets[i];
        let href = stylesheet.href;

        // fetch external stylesheet
        if (href) {
        await fetchExternalStylesheet(href, i);
        } else {
        let cssRules = stylesheet.cssRules;
        let cssRulesL = cssRules.length;
        for (let r = 0; r < cssRulesL; r++) {
            let rule = cssRules[r];
            let type = rule.type;

            // is @import rule - fetch external stylesheet
            if (type === 3) {
            let href = rule.href;
            if (href) {
                if (removeExternalCss) {
                stylesheet.deleteRule(r);
                cssRulesL--;
                r--;
                }
                await fetchExternalStylesheet(href, r);
            }
            } else if (type === 5) {
            origins.push("local");
            }
        }
        }
    }

    // remove external stylesheet
    if (removeExternalCss) {
        let styleSheetLinks = document.querySelectorAll(
        'link[href$=".css"], link[rel="stylesheet"]'
        );
        for (let i = 0; i < styleSheetLinks.length; i++) {
        let link = styleSheetLinks[i];
        await link.remove();
        }
    }

    let styleSheetsInlined = Array.from(document.styleSheets);
    return styleSheetsInlined;
    }
     async function getFontData() {
        let fontRefs=await getFontReferences()
        let fontURL=""
        for(let f of fontRefs) {
            if (layer.font===f.fontFamily) {
                fontURL=f.src
            }
        }
        if (!fontURL)  return 
        let resp = await fetch(fontURL);
        let blob = await resp.blob()
        return new Promise((resolve) => {
            let f = new FileReader()
            f.addEventListener('load', () => resolve(f.result))
            f.readAsDataURL(blob);
        })
    }
    export async function renderForAI() {
        await _renderForAI()    // twice because of iPad bug
//        await  new Promise(resolve => setTimeout(resolve, 300));
        return await _renderForAI()
    }
    export async function _renderForAI() {
                // first load the font
        let fontData=await getFontData()
        let style=""
        if (fontData) {
            style="<style>\n @font-face { font-family: '"+layer.font+"';"
            style+="\nsrc: url("+fontData+") format('woff2');\n</style> "
        }
           
        let svgData = new XMLSerializer().serializeToString(svg)
        svgData=svgData.replace("<defs/>","<defs>"+style+"</defs>")   // embed font

        const svgDataBase64 = btoa(unescape(encodeURIComponent(svgData)))        
        const svgDataUrl = `data:image/svg+xml;charset=utf-8;base64,${svgDataBase64}`        
        
        const canvas = document.createElement('canvas')
        let image = new Image()
        return new Promise( (resolve) =>  {
            image.addEventListener('load',async () => {
                canvas.width=parseInt( width)
                canvas.height=parseInt( height)
                const context = canvas.getContext('2d')
                context.drawImage(image, 0, 0, width, height)
                const dataUrl = canvas.toDataURL('image/png')
                resolve(dataUrl)
            })
            image.src = svgDataUrl
        })

    }
        



   
    export let layer={string:"Hello",font_size:150,align:"middle",background:"transparent",bold:false,italic:false,underline:false} 
    /* aditional layer data: for every plugin
        element: point to DOM tree element which contains this component
        data:    exported serialized data of this layer
        x,y,width,height: size and position of this layer. both can be changed by move-tool
        _x,_y,_width,_height: scaled cooedinates
        url: temporary image data
        name: the layer name which can be changed by user
        id: internal ID
    */ 
    let fontweight="normal";
    let fontstyle="normal";
    let textdecoration="none";

    $: {        
        if (layer) {
            if (!layer.align) layer.align="middle"
            layer=layer
        }
        if(layer && layer.bold){
            fontweight = "bold";
        }  else{
            fontweight = "normal";
        }

        if(layer && layer.italic){
            fontstyle="italic";
        }else{
            fontstyle="normal";
        }

        if(layer && layer.underline){
            textdecoration="underline";
        }else{
            textdecoration="none";
        }



        if (layer && layer.string) {
            calcText()
        }
        if (layer && layer.font_size) {
            calcText()
        }
        if (width || height) {
            calcInnerSize()
        }
    }
    let innerHeight=1000
    let innerWidth=1000
    function calcInnerSize() {
        let ar=width/height
        if (width>height) {
            innerHeight=1000/ar
            innerWidth=1000
        } else {
            innerWidth=1000/ar
            innerHeight=1000
        }
    }
    let lines=[]
    let lineHeight
    let yStart
    function calcText() {
        if  (!layer.string) return 
        lines=layer.string.split("\n")
        lineHeight=layer.font_size*1.2
        yStart=-(lineHeight*(lines.length-1))/2
    }

    export async function autoSize() {
        let t=host.shadowRoot.querySelector("#text")
        if (!t) return

        let rect=t.getBoundingClientRect()
        if (rect.width>=width) {
            while (rect.width>width) {
                layer.font_size-=10
                await new Promise(resolve => setTimeout(resolve, 1))
                rect=t.getBoundingClientRect()
            }
            return
        }
        while (rect.width<width) {
            layer.font_size+=10
            await new Promise(resolve => setTimeout(resolve, 1))
            rect=t.getBoundingClientRect()
        }
        layer.font_size-=10
    }

    /**
     * get full scene data: camera and all poses
     */
    export function getScene() {
        let scene={  text:layer.string, font_size:layer.font_size,font:layer.font,align:layer.align,background:layer.background}
        let sceneData=JSON.parse(JSON.stringify(scene))
        return sceneData
    }
    /**
     * set scene and pose data
     */
     export async function setScene(sceneInfo) {
        layer.string=sceneInfo.text
        layer.font_size=sceneInfo.font_size
        layer.font=sceneInfo.font
        layer.align=sceneInfo.align
        layer.background=sceneInfo.background
    }

    /**
     * make image from current scene 
     * @return: base64-url
     */
    export async function renderNormal() {
//        let url=canvas.toDataURL()             
//        setMode(oldMode)
let url=""
        return url
    }
   


  
    
    /**
     * canvas can be used somewhere else
     */
    export let svg
</script>
<svg {width}  {height} style="background:{layer.background}"  viewBox="0 0 {innerWidth} {innerHeight}" bind:this={svg}>
    <defs></defs>

    <text id="text" x="0" y="{innerHeight/2}"  style="text-decoration:{textdecoration}; font-style:{fontstyle}; font-weight:{fontweight}; font-size:{layer.font_size}px;font-family:'{layer.font}';letter-spacing:{layer.letter_spacing}rem"  dominant-baseline="middle" >
        {#each lines as text,i}       
            <tspan x="{innerWidth/2}" y="{innerHeight/2}" dy="{yStart+lineHeight*i}px" text-anchor="{layer.align}">{text}</tspan>
        {/each}
    </text>
  </svg>
<style>

    * {
        box-sizing: border-box;
    }
    /** delete this CSS: */
</style>
