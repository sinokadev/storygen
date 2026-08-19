<script>
    import { onMount, onDestroy } from 'svelte';
    import { Editor, Mark, mergeAttributes } from '@tiptap/core';
    import StarterKit from '@tiptap/starter-kit';
    import { TextStyle } from '@tiptap/extension-text-style';
    import { Color } from '@tiptap/extension-color';
    import { Highlight } from '@tiptap/extension-highlight';
    import { toPng } from 'html-to-image';

    // 1. 커스텀 FontSize Mark Extension 정의
    const FontSize = Mark.create({
        name: 'fontSize',

        addOptions() {
            return {
                types: ['textStyle'],
            };
        },

        addAttributes() {
            return {
                size: {
                    default: null,
                    parseHTML: element => element.style.fontSize?.replace(/['"]+/g, '') || null,
                    renderHTML: attributes => {
                        if (!attributes.size) {
                            return {};
                        }
                        return {
                            style: `font-size: ${attributes.size}`,
                        };
                    },
                },
            };
        },

        parseHTML() {
            return [
                {
                    style: 'font-size',
                },
            ];
        },

        renderHTML({ HTMLAttributes }) {
            return ['span', mergeAttributes(this.options.HTMLAttributes, HTMLAttributes), 0];
        },

        addCommands() {
            return {
                setFontSize: size => ({ chain }) => {
                    return chain().setMark(this.name, { size }).run();
                },
                unsetFontSize: () => ({ chain }) => {
                    return chain().unsetMark(this.name).removeEmptyTextStyle().run();
                },
            };
        },
    });

    // 날짜 기본값 세팅
    let today = new Date();   
    let year = today.getFullYear();
    let month = today.getMonth() + 1;
    let date = today.getDate();

    // 상태 관리 (Svelte 5 Runes)
    let position = $state('center-center');
    let currentColor = $state('#000000');
    let currentBgColor = $state('#ffec3d'); 
    let thumbnailBgColor = $state('#ffffff'); // 썸네일 전체 배경색
    let thumbnailBgImage = $state(''); // 썸네일 배경 이미지 (Data URL)
    let isDraggingOver = $state(false); // 드래그 앤 드롭 상태 감지
    let selectedFontSize = $state('default');
    
    // 폰트 관련 상태
    let selectedFontFamily = $state('Pretendard, sans-serif');
    let customFontInput = $state('');

    let editorElement = $state(null);
    let editor = $state(null);
    let thumbnailRef = $state(null);
    let fileInputRef = $state(null);

    // 프리셋 폰트 목록
    const fontPresets = [
        { label: 'Pretendard (기본)', value: 'Pretendard, sans-serif' },
        { label: '고딕 / Sans-serif (System)', value: 'system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif' },
        { label: '명조 / Serif', value: '"Times New Roman", Times, Georgia, serif' },
        { label: '나눔고딕 (Nanum Gothic)', value: '"Nanum Gothic", sans-serif' },
        { label: '나눔명조 (Nanum Myeongjo)', value: '"Nanum Myeongjo", serif' },
        { label: '궁서체 (Gungsuh)', value: 'Gungsuh, serif' },
        { label: 'Arial', value: 'Arial, sans-serif' },
        { label: 'Impact (강렬함)', value: 'Impact, sans-serif' },
        { label: 'Comic Sans MS', value: '"Comic Sans MS", cursive, sans-serif' },
        { label: 'Courier New (고정폭)', value: '"Courier New", monospace' }
    ];

    // 선택할 수 있는 텍스트 크기 옵션 목록
    const fontSizes = [
        '0.75rem', '0.875rem', '1rem', '1.25rem', '1.5rem', 
        '1.75rem', '2rem', '2.5rem', '3rem', '3.5rem', '4rem', '5rem', '6rem'
    ];

    // 선택된 커서/영역의 스타일 상태를 툴바 UI에 동기화
    function updateToolbarState() {
        if (!editor) return;

        const fontSizeAttr = editor.getAttributes('fontSize').size 
                          || editor.getAttributes('textStyle').size;

        if (fontSizeAttr) {
            selectedFontSize = fontSizeAttr;
        } else {
            const activeMarks = editor.state.selection.$from.marks();
            const fontSizeMark = activeMarks.find(m => m.type.name === 'fontSize' || m.attrs.size);
            selectedFontSize = fontSizeMark?.attrs?.size || 'default';
        }

        const textColor = editor.getAttributes('textStyle').color;
        currentColor = textColor || '#000000';

        const highlightColor = editor.getAttributes('highlight').color;
        currentBgColor = highlightColor || '#ffec3d';
    }

    onMount(() => {
        editor = new Editor({
            element: editorElement,
            extensions: [
                StarterKit,
                TextStyle,
                FontSize,
                Color,
                Highlight.configure({ multicolor: true })
            ],
            content: `<p>${year}. ${month}. ${date}.</p>`,
            onTransaction: () => {
                editor = editor;
                updateToolbarState(); 
            },
            onSelectionUpdate: () => {
                updateToolbarState();
            }
        });
    });

    onDestroy(() => {
        if (editor) {
            editor.destroy();
        }
    });

    // Tiptap 서식 제어 함수들
    function toggleBold() {
        editor?.chain().focus().toggleBold().run();
    }

    function toggleItalic() {
        editor?.chain().focus().toggleItalic().run();
    }

    function toggleStrike() {
        editor?.chain().focus().toggleStrike().run();
    }

    function changeColor(e) {
        const color = e.target.value;
        currentColor = color;
        editor?.chain().focus().setColor(color).run();
    }

    function changeHighlightColor(e) {
        const color = e.target.value;
        currentBgColor = color;
        editor?.chain().focus().setHighlight({ color }).run();
    }

    function changeThumbnailBg(e) {
        thumbnailBgColor = e.target.value;
    }

    function changeFontSize(e) {
        const size = e.target.value;
        selectedFontSize = size;

        if (size === 'default') {
            editor?.chain().focus().unsetFontSize().run();
        } else {
            editor?.chain().focus().setFontSize(size).run();
        }
    }

    // 폰트 셀렉트 및 커스텀 입력 제어
    function handleFontSelect(e) {
        const value = e.target.value;
        if (value !== 'custom') {
            selectedFontFamily = value;
            customFontInput = '';
        }
    }

    function handleCustomFontInput(e) {
        const value = e.target.value;
        customFontInput = value;
        if (value.trim()) {
            selectedFontFamily = value;
        } else {
            selectedFontFamily = 'Pretendard, sans-serif';
        }
    }

    function clearFormatting() {
        editor?.chain().focus().unsetAllMarks().run();
        selectedFontSize = 'default';
        currentColor = '#000000';
        currentBgColor = '#ffec3d';
    }

    // 이미지 파일을 읽어서 Data URL로 로드하는 공통 함수
    function processImageFile(file) {
        if (!file || !file.type.startsWith('image/')) return;

        const reader = new FileReader();
        reader.onload = () => {
            if (typeof reader.result === 'string') {
                thumbnailBgImage = reader.result;
            }
        };
        reader.readAsDataURL(file);
    }

    // 파일 선택(클릭) 업로드 핸들러
    function handleImageUpload(e) {
        const file = e.target.files?.[0];
        processImageFile(file);
    }

    // 드래그 앤 드롭 이벤트 핸들러들
    function handleDragOver(e) {
        e.preventDefault();
        isDraggingOver = true;
    }

    function handleDragLeave(e) {
        e.preventDefault();
        isDraggingOver = false;
    }

    function handleDrop(e) {
        e.preventDefault();
        isDraggingOver = false;

        const file = e.dataTransfer?.files?.[0];
        processImageFile(file);
    }

    // 배경 이미지 제거 함수
    function removeBgImage() {
        thumbnailBgImage = '';
        if (fileInputRef) {
            fileInputRef.value = '';
        }
    }

    // 썸네일 이미지 다운로드
    async function downloadThumbnail() {
        if (!thumbnailRef) return;

        try {
            const dataUrl = await toPng(thumbnailRef, { 
                cacheBust: true,
                style: {
                    borderColor: 'transparent'
                }
            });
            
            const link = document.createElement('a');
            link.download = `thumbnail-${Date.now()}.png`;
            link.href = dataUrl;
            link.click();
        } catch (err) {
            console.error('캡처 중 오류가 발생했습니다:', err);
        }
    }
</script>


  <header>
    <h1>섬네일 생성기</h1>
    <p>스토리 서버의 섬네일 공모전에 제출할 섬네일이나 기타 섬네일을 생성할 수 있는 페이지입니다. by <a href="https://sinoka.dev">SinokaDev🧊</a></p>
    
  </header>

<main>
    <!-- Tiptap 서식 툴바 -->
    {#if editor}
        <div class="toolbar">
            <button 
                type="button"
                class:active={editor.isActive('bold')} 
                onclick={toggleBold}
                title="굵게">
                <b>B</b>
            </button>

            <button 
                type="button"
                class:active={editor.isActive('italic')} 
                onclick={toggleItalic}
                title="기울임꼴">
                <i>I</i>
            </button>

            <button 
                type="button"
                class:active={editor.isActive('strike')} 
                onclick={toggleStrike}
                title="취소선">
                <s>S</s>
            </button>

            <!-- 폰트 드롭다운 및 직접 입력 -->
            <select 
                value={customFontInput ? 'custom' : selectedFontFamily} 
                onchange={handleFontSelect} 
                class="font-select" 
                title="폰트 패밀리 선택">
                {#each fontPresets as font}
                    <option value={font.value}>{font.label}</option>
                {/each}
                <option value="custom">직접 입력...</option>
            </select>

            <input 
                type="text" 
                value={customFontInput} 
                oninput={handleCustomFontInput} 
                placeholder="예: 맑은 고딕" 
                class="custom-font-input" 
                title="시스템/CSS 폰트명 직접 입력" />

            <select value={selectedFontSize} onchange={changeFontSize} class="font-size-select" title="글자 크기">
                <option value="default">크기 기본값</option>
                {#each fontSizes as size}
                    <option value={size}>{size}</option>
                {/each}
            </select>

            <label class="color-picker-label" title="글자 색상">
                🎨
                <input 
                    type="color" 
                    value={currentColor} 
                    oninput={changeColor} />
            </label>

            <label class="color-picker-label" title="텍스트 배경색 (형광펜)">
                🖍️
                <input 
                    type="color" 
                    value={currentBgColor} 
                    oninput={changeHighlightColor} />
            </label>

            <label class="color-picker-label" title="썸네일 전체 배경색">
                🖼️
                <input 
                    type="color" 
                    value={thumbnailBgColor} 
                    oninput={changeThumbnailBg} />
            </label>

            <button type="button" onclick={clearFormatting} title="서식 초기화" class="clear-btn">
                🧹
            </button>
        </div>
    {/if}

    <!-- 썸네일 영역 -->
    <div 
        id="thumbnail" 
        bind:this={thumbnailRef} 
        class="{position}"
        class:drag-over={isDraggingOver}
        ondragover={handleDragOver}
        ondragleave={handleDragLeave}
        ondrop={handleDrop}
        style="
            background-color: {thumbnailBgColor};
            background-image: {thumbnailBgImage ? `url(${thumbnailBgImage})` : 'none'};
        ">
        <div 
            class="in-thumbnail-text" 
            bind:this={editorElement}
            style="font-family: {selectedFontFamily};">
        </div>
        {#if isDraggingOver}
            <div class="drag-overlay">이미지를 여기에 놓으세요</div>
        {/if}
    </div>

    <!-- 배경 이미지 제어 컨트롤 (툴바 외부) -->
    <div class="bg-image-controls">
        <input 
            type="file" 
            accept="image/*" 
            bind:this={fileInputRef} 
            onchange={handleImageUpload} 
            id="bg-file-input" 
            class="hidden-file-input" />
        
        <label for="bg-file-input" class="bg-btn add-bg-btn">
            🖼️ 배경 이미지 추가
        </label>

        {#if thumbnailBgImage}
            <button type="button" onclick={removeBgImage} class="bg-btn remove-bg-btn">
                🗑️ 배경 이미지 제거
            </button>
        {/if}
    </div>

    <!-- 정렬 컨트롤 -->
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

    <button type="button" class="download-btn" onclick={downloadThumbnail}>
        섬네일 다운로드
    </button>
</main>

<footer>
    이 페이지의 기본 폰트는 Pretendard입니다. 섬네일의 기본 폰트 또한 Pretendard입니다. <br>
    <a href="https://github.com/orioncactus/pretendard" target="_blank" rel="noreferrer">https://github.com/orioncactus/pretendard</a> <br>
    섬네일 생성기 by <a href="https://sinoka.dev">SinokaDev🧊</a>
</footer>

<style>
    main {
        display: flex;
        flex-direction: column;
        align-items: center;
        text-align: center;
        gap: 12px;
        max-width: 800px;
        margin: 0 auto;
        padding: 20px;
    }

    /* 툴바 메인 컨테이너 */
    .toolbar {
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        gap: 6px;
        padding: 6px 10px;
        background: #ffffff;
        border: 1px solid #e1e4e8;
        border-radius: 8px;
        box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
    }

    /* 공통 버튼 & 선택창 스타일 */
    .toolbar button, 
    .toolbar select,
    .custom-font-input {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        height: 34px;
        padding: 0 10px;
        border: 1px solid #d1d5db;
        background: #ffffff;
        border-radius: 6px;
        color: #374151;
        font-size: 0.875rem;
        font-weight: 500;
        cursor: pointer;
        transition: all 0.15s ease-in-out;
    }

    .toolbar button:hover, 
    .toolbar select:hover {
        background: #f3f4f6;
        border-color: #9ca3af;
    }

    .toolbar button:active {
        transform: translateY(1px);
    }

    .toolbar button.active {
        background: #eff6ff;
        color: #2563eb;
        border-color: #3b82f6;
    }

    .font-select {
        max-width: 160px;
        outline: none;
    }

    .custom-font-input {
        width: 140px;
        cursor: text;
        outline: none;
    }
    .custom-font-input:focus {
        border-color: #4f46e5;
        box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
    }

    .font-size-select {
        padding-right: 12px;
        outline: none;
    }

    .color-picker-label {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        gap: 4px;
        padding: 0 8px;
        height: 34px;
        border: 1px solid #d1d5db;
        border-radius: 6px;
        background: #ffffff;
        cursor: pointer;
        font-size: 0.875rem;
        transition: all 0.15s ease-in-out;
    }

    .color-picker-label:hover {
        background: #f3f4f6;
        border-color: #9ca3af;
    }

    .color-picker-label input[type="color"] {
        -webkit-appearance: none;
        -moz-appearance: none;
        appearance: none;
        width: 18px;
        height: 18px;
        border: none;
        border-radius: 50%;
        cursor: pointer;
        padding: 0;
        background: none;
    }

    .color-picker-label input[type="color"]::-webkit-color-swatch-wrapper {
        padding: 0;
    }

    .color-picker-label input[type="color"]::-webkit-color-swatch {
        border: 1px solid rgba(0, 0, 0, 0.15);
        border-radius: 50%;
    }

    .thumbnail-bg-label {
        border-color: #6366f1;
        background: #f5f3ff;
    }
    .thumbnail-bg-label:hover {
        background: #ede9fe;
    }

    .clear-btn:hover {
        background: #fef2f2;
        border-color: #f87171;
        color: #ef4444;
    }

    /* 썸네일 메인 컨테이너 */
    #thumbnail {
        box-sizing: border-box;
        width: 100%;
        aspect-ratio: 16 / 9;
        padding: 2em;
        border-radius: 5px;
        border: 2px solid black;

        display: grid;
        place-items: center; 
        place-content: center;

        overflow: hidden; 
        position: relative;

        background-size: cover;
        background-position: center;
        background-repeat: no-repeat;
        transition: border-color 0.2s ease, background-color 0.2s ease;
    }

    /* 드래그 오버 상태 스타일 */
    #thumbnail.drag-over {
        border: 2px dashed #4f46e5;
    }

    .drag-overlay {
        position: absolute;
        inset: 0;
        background: rgba(79, 70, 229, 0.2);
        backdrop-filter: blur(2px);
        display: flex;
        align-items: center;
        justify-content: center;
        color: #312e81;
        font-weight: bold;
        font-size: 1.25rem;
        pointer-events: none;
    }

    /* 배경 이미지 전용 버튼 컨테이너 */
    .bg-image-controls {
        display: flex;
        gap: 8px;
        margin-top: 4px;
    }

    .hidden-file-input {
        display: none;
    }

    .bg-btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        padding: 8px 14px;
        font-size: 0.875rem;
        font-weight: 600;
        border-radius: 6px;
        cursor: pointer;
        transition: all 0.15s ease-in-out;
    }

    .add-bg-btn {
        background-color: #4f46e5;
        color: white;
        border: none;
    }

    .add-bg-btn:hover {
        background-color: #4338ca;
    }

    .remove-bg-btn {
        background-color: #dc2626;
        color: white;
        border: none;
    }

    .remove-bg-btn:hover {
        background-color: #b91c1c;
    }

    /* Tiptap 에디터 요소 내부 스타일 설정 */
    .in-thumbnail-text {
        color: inherit;
        width: max-content;
        height: max-content;
        max-width: 100%;
        font-size: 2.5rem;
    }

    .in-thumbnail-text :global(.ProseMirror) {
        outline: none;
        overflow: hidden;
        overflow-anchor: none;
    }

    .in-thumbnail-text :global(.ProseMirror p) {
        margin: 0;
        line-height: 1.2;
    }

    /* 9가지 정렬 위치 설정 */
    #thumbnail.top-left       { place-content: start start; }
    #thumbnail.top-center     { place-content: start center; }
    #thumbnail.top-right      { place-content: start end; }

    #thumbnail.center-left    { place-content: center start; }
    #thumbnail.center-center  { place-content: center center; }
    #thumbnail.center-right   { place-content: center end; }

    #thumbnail.bottom-left    { place-content: end start; }
    #thumbnail.bottom-center  { place-content: end center; }
    #thumbnail.bottom-right   { place-content: end end; }

    /* 텍스트 내부 정렬 동기화 */
    #thumbnail.top-left .in-thumbnail-text,
    #thumbnail.center-left .in-thumbnail-text,
    #thumbnail.bottom-left .in-thumbnail-text {
        text-align: left;
    }

    #thumbnail.top-center .in-thumbnail-text,
    #thumbnail.center-center .in-thumbnail-text,
    #thumbnail.bottom-center .in-thumbnail-text {
        text-align: center;
    }

    #thumbnail.top-right .in-thumbnail-text,
    #thumbnail.center-right .in-thumbnail-text,
    #thumbnail.bottom-right .in-thumbnail-text {
        text-align: right;
    }

    #position-select {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        grid-template-rows: repeat(3, 1fr);
        width: 100px;
        height: 100px;
        gap: 4px;
    }

    .download-btn {
        padding: 10px 20px;
        font-size: 1rem;
        font-weight: bold;
        background-color: #28a745;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
    }

    .download-btn:hover {
        background-color: #218838;
    }
</style>