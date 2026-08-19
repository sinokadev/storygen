
<script>

</script>

<main>
    <div id="thumbnail">
        <div>2026. 10. 06.</div>
    </div>
    <input type="text" placeholder="섬네일에 들어갈 텍스트를 입력하세요...">

    <p>텍스트 정렬</p>
    <div id="position-select">
        <input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox"><input type="checkbox">
    </div>
</main>

<style>
    main {
        display: flex !important;
        flex-direction: column !important;
        align-items: center !important;
        text-align: center !important;
    }
    #thumbnail {
        box-sizing: border-box; /* 패딩과 테두리를 width 내부로 포함 */
        width: 100%;
        aspect-ratio: 16 / 9;
        object-fit: cover;
        padding: 1em;
        border-radius: 5px;
        border-style: solid;
        border-width: 2px;
        border-color: black;
    }
    #position-select {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        grid-template-rows: repeat(3, 1fr);
        width: 100px;
        height: 100px;
    }
</style>