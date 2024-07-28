<svelte:options tag="fds-image-editor-text-toolbar"></svelte:options>
<script>
    import {get_current_component} from "svelte/internal"
    import {clickOutside} from "./clickOutside";

    let fonts = ["Montez", "Lobster", "Josefin Sans", "Shadows Into Light", "Pacifico", "Amatic SC", "Orbitron", "Rokkitt", "Righteous", "Dancing Script", "Bangers", "Chewy", "Sigmar One", "Architects Daughter", "Abril Fatface", "Covered By Your Grace", "Kaushan Script", "Gloria Hallelujah", "Satisfy", "Lobster Two", "Comfortaa", "Cinzel", "Courgette"];
    let host = get_current_component()
    let dropdownel;
    export let values = {
        selectedLayer: {
            font_size: 150,
            letter_spacing: 1,
            bold: false,
            italic: false,
            underline: false
        }
    }

    function action(name, value) {
        host.dispatchEvent(new CustomEvent("action", {detail: {name: name, value: value}}))
    }

    function change(name, value) {
        host.dispatchEvent(new CustomEvent("change", {detail: {name: name, value: value}}))
    }

    function handleClickOutside() {
        dropdownel.classList.remove("show");
    }

    function toggleDropdownVisibility() {
        dropdownel.classList.toggle("show");
    }

    $: if (values) {
        values = values
    }

    let editText
    export function refresh() {
        values = values
    }
</script>

<div class="show">
    {#if values.selectedLayer}
        <div style="display:flex;align-items: center">
            Text |
            Font Size: <span class="values3" style="display: inline-block; margin-right: 10px">{values.selectedLayer.font_size}</span>
            <fds-image-editor-slider
                    class="slider"
                    min="50"
                    max="900"
                    on:input={(e) => { values.selectedLayer.font_size=e.target.value;values=values;change("change","font_size") }}
                    value={values.selectedLayer.font_size}
                    style="margin-right:20px"
            ></fds-image-editor-slider>

            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <fds-image-editor-button type="button" on:click={() => { editText=!editText }}>Text/Font...</fds-image-editor-button>
        </div>
    {/if}

    {#if editText && values.selectedLayer }
        <div class="editText">

            <div class="dropdown" use:clickOutside on:click_outside={handleClickOutside}>
                <button
                        class="font-preview"
                        style="font-family:{values.selectedLayer.font}"
                        on:click={() => {toggleDropdownVisibility() }}
                >{values.selectedLayer.font}</button>
                <div id="myDropdown" bind:this={dropdownel} class="dropdown-content">
                    {#each fonts as font}
                        <a on:click={(e) => {  values.selectedLayer.font=e.target.text;values=values;change("change","font") }} href="#"
                           style="font-family:{font}">{font}</a>
                    {/each}
                </div>
            </div>

            <div style="margin-top: 16px">
                Letter Spacing: <span class="values3">{values.selectedLayer.letter_spacing}</span>
            </div>
            <div style="margin-top: 6px">
                <fds-image-editor-slider
                        min="-2"
                        max="8"
                        step="0.1"
                        on:input={(e) => { values.selectedLayer.letter_spacing=e.target.value;values=values;change("change","letter_spacing") }}
                        value={values.selectedLayer.letter_spacing}
                        style=""
                ></fds-image-editor-slider>
            </div>
            <div class="buttoncontainer">
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <fds-image-editor-button
                        icon="bold"
                        title="Bold"
                        on:click={() => { values.selectedLayer.bold=!values.selectedLayer.bold;change("change","font_weight") }}
                        state={values.selectedLayer.bold ? 'active' : ''}
                ></fds-image-editor-button>
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <fds-image-editor-button
                        icon="italic"
                        title="Italic"
                        on:click={() => { values.selectedLayer.italic=!values.selectedLayer.italic;change("change","font_style"); }}
                        state={values.selectedLayer.italic ? 'active' : ''}
                ></fds-image-editor-button>
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <fds-image-editor-button
                        icon="underline"
                        title="Underline"
                        on:click={() => {values.selectedLayer.underline=!values.selectedLayer.underline;change("change","text_decoration"); }}
                        state={values.selectedLayer.underline ? 'active' : ''}
                ></fds-image-editor-button>
            </div>

            <div class="formLine">
                <textarea class="formInput" style="height:100px;font-size:20px;font-family:{values.selectedLayer.font}"
                          on:change={(e) => { values.selectedLayer.string=e.target.value;values=values;change("change","text") }}>{values.selectedLayer.string}</textarea>
            </div>
        </div>
    {/if}
</div>

<style>
    * {
        box-sizing: border-box;
    }

    .buttoncontainer {
        margin-top: 10px;
        margin-bottom: 10px;
        padding-bottom: 5px;;
        border-bottom: 1px solid white;
    }

    button {
        color: white;
        background-color: transparent;
    }

    .editText {
        min-width: 200px;
        min-height: 100px;
        box-shadow: 0 0 1rem 0 rgba(255, 255, 255, .2);
        border-radius: 8px;
        background-color: rgba(0, 0, 0, .65);
        backdrop-filter: blur(10px);
        color: rgba(255, 255, 255, 0.6);
        position: absolute;
        left: 150px;
        top: 50px;
        padding: 14px;
        font-family: "Segoe UI", Roboto, system-ui;
        z-index: 2000;
        margin-left: 100px;
    }

    .formLine {
        displaY: block;
        margin-bottom: 20px;
    }

    .slider {
        vertical-align: middle;
    }

    .values3 {
        width: 30px;
        text-align: right;
        display: inline-block;
    }

    .formInput {
        width: 100%;
        outline: 0;
        border: 0;
        border-bottom: 2px solid rgba(255, 255, 255, 0.2);
        padding: 5px;
        font-size: 14px;
        font-weight: normal;
        border-radius: 3px;
        color: rgba(255, 255, 255, 0.9);
        margin-bottom: 10px;
        display: block;
        background: none;
        font-family: system-ui, "Segoe UI", Roboto;

    }

    .formInput::placeholder {
        font-weight: normal;
        opacity: 0.5;
        color: rgba(255, 255, 255, 0.7);
    }

    .formLine {
        margin-bottom: 5px;
    }

    .dropdown {
        position: relative;
        display: inline-block;
    }

    .dropdown-content {
        display: none;
        position: absolute;
        background-color: #f1f1f1;
        min-width: 160px;
        overflow: auto;
        box-shadow: 0 8px 16px 0 rgba(0, 0, 0, 0.2);
        z-index: 1;
        max-height: calc(100vh / 2);
        margin-top: 5px;
        border-radius: 6px;
    }

    .dropdown-content a {
        color: black;
        padding: 12px 16px;
        text-decoration: none;
        display: block;
        font-size: 20px;
    }

    .dropdown a:hover {
        background-color: #ddd;
    }

    .show {
        display: block;
    }

    .font-preview {
        display:inline-block;
        width:200px;
        color:black;
        background-color:white;
        font-size:20px;
        padding: 16px;
        border: none;
        cursor: pointer;
        border-radius: 6px;
    }

</style>