<script>
    let thumtext = $state('world');

    let today = new Date();   
    let year = today.getFullYear();
    let month = today.getMonth() + 1;
    let date = today.getDate();

    thumtext = `${year}. ${month}. ${date}.`;

    let textsize = $state('5rem');
    let position = $state('center-center');
</script>

<main>
    <!-- class={position} 및 style:--textsize={textsize} 연결 -->
    <div id="thumbnail" class={position} style:--textsize={textsize}>
        <div id="thumtext">{thumtext}</div>
    </div>
    
    <input type="text" placeholder="섬네일에 들어갈 텍스트를 입력하세요..." bind:value={thumtext}>

    <p>텍스트 정렬</p>
    <div id="position-select">
        <input type="radio" bind:group={position} value="top-left" />
        <input type="radio" bind:group={position} value="top-center" />
        <input type="radio" bind:group={position} value="top-right" />

        <input type="radio" bind:group={position} value="center-left" />
        <input type="radio" bind:group={position} value="center-center" />
        <input type="radio" bind:group={position} value="center-right" />

        <input type="radio" bind:group={position} value="bottom-left" />
        <input type="radio" bind:group={position} value="bottom-center" />
        <input type="radio" bind:group={position} value="bottom-right" />
    </div>
</main>

<style>
    main {
        display: flex !important;
        flex-direction: column !important;
        align-items: center !important;
        text-align: center !important;
        gap: 10px;
    }
    #thumtext {
        font-size: var(--textsize);
    }
    #thumbnail {
        box-sizing: border-box;
        width: 100%;
        aspect-ratio: 16 / 9;
        padding: 1em;
        border-radius: 5px;
        border: 2px solid black;

        display: flex;
        justify-content: center;
        align-items: center;
    }
    #position-select {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        grid-template-rows: repeat(3, 1fr);
        width: 100px;
        height: 100px;
    }

    /* 위치 클래스들 */
    #thumbnail.top-left       { justify-content: flex-start; align-items: flex-start; }
    #thumbnail.top-center     { justify-content: center;     align-items: flex-start; }
    #thumbnail.top-right      { justify-content: flex-end;   align-items: flex-start; }
    
    #thumbnail.center-left    { justify-content: flex-start; align-items: center; }
    #thumbnail.center-center  { justify-content: center;     align-items: center; }
    #thumbnail.center-right   { justify-content: flex-end;   align-items: center; }
    
    #thumbnail.bottom-left    { justify-content: flex-start; align-items: flex-end; }
    #thumbnail.bottom-center  { justify-content: center;     align-items: flex-end; }
    #thumbnail.bottom-right   { justify-content: flex-end;   align-items: flex-end; }
</style>