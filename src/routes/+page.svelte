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

    // padStart(최종길이, 채울문자) 사용
    let month = String(today.getMonth() + 1).padStart(2, '0');
    let date = String(today.getDate()).padStart(2, '0');

    // 상태 관리 (Svelte 5 Runes)
    let position = $state('center-center');
    let currentColor = $state('#000000');
    let currentBgColor = $state('#ffec3d'); 
    let thumbnailBgColor = $state('#ffffff'); // 썸네일 전체 배경색
    let thumbnailBgImage = $state(''); // 썸네일 배경 이미지 (Data URL)
    let isDraggingOver = $state(false); // 드래그 앤 드롭 상태 감지
    let selectedFontSize = $state('default');
    let thumbnailTextColor = $state('#000000');
    
    // 폰트 관련 상태
    let selectedFontFamily = $state('Pretendard, sans-serif');
    let customFontInput = $state('');
    let editorElement = $state(null);
    let editor = $state(null);
    let thumbnailRef = $state(null);
    let fileInputRef = $state(null);
    let jsonFileInputRef = $state(null); // JSON 파일 업로드용 참조

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

        const fontSizeAttr =
            editor.getAttributes('fontSize').size ||
            editor.getAttributes('textStyle').size;

        if (fontSizeAttr) {
            selectedFontSize = fontSizeAttr;
        } else {
            const activeMarks = editor.state.selection.$from.marks();
            const fontSizeMark = activeMarks.find(
                m => m.type.name === 'fontSize' || m.attrs.size
            );

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

    // ==========================================
    // JSON 데이터 저장 (내보내기) 기능
    // ==========================================
    function exportJson() {
        if (!editor) return;

        const thumbnailData = {
            version: 1,
            position,
            thumbnailBgColor,
            thumbnailBgImage,
            selectedFontFamily,
            customFontInput,
            editorContent: editor.getJSON() // Tiptap 내부 문단/서식 데이터를 JSON 구조로 추출
        };

        const jsonString = JSON.stringify(thumbnailData, null, 2);
        const blob = new Blob([jsonString], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        
        const a = document.createElement('a');
        a.href = url;
        a.download = `thumbnail-preset-${Date.now()}.json`;
        a.click();
        URL.revokeObjectURL(url);
    }

    // ==========================================
    // JSON 데이터 불러오기 (적용) 기능
    // ==========================================
    function importJson(e) {
        const file = e.target.files?.[0];
        if (!file) return;

        const reader = new FileReader();
        reader.onload = (event) => {
            try {
                const data = JSON.parse(event.target.result);

                // 상태 데이터 복원
                position = data.position || 'center-center';
                thumbnailBgColor = data.thumbnailBgColor || '#ffffff';
                thumbnailBgImage = data.thumbnailBgImage || '';
                selectedFontFamily = data.selectedFontFamily || 'Pretendard, sans-serif';
                customFontInput = data.customFontInput || '';

                // Tiptap 에디터 내용/서식 데이터 복원
                if (data.editorContent && editor) {
                    editor.commands.setContent(data.editorContent);
                }
            } catch (err) {
                alert('올바른 JSON 파일이 아닙니다.');
                console.error('JSON 로드 중 오류 발생:', err);
            } finally {
                // 동일 파일 재업로드를 위해 input 비우기
                if (jsonFileInputRef) jsonFileInputRef.value = '';
            }
        };
        reader.readAsText(file);
    }

    // 썸네일 이미지 다운로드
async function downloadThumbnail() {
    if (!thumbnailRef) return;

    try {
        const targetWidth = 1920;
        const targetHeight = 1080;

        const currentWidth = thumbnailRef.offsetWidth;

        // 16:9이므로 하나의 scale만 사용
        const scale = targetWidth / currentWidth;

        const dataUrl = await toPng(thumbnailRef, {
            cacheBust: true,
            width: targetWidth,
            height: targetHeight,
            canvasWidth: targetWidth,
            canvasHeight: targetHeight,

            style: {
                transform: `scale(${scale})`,
                transformOrigin: 'top left',
                width: `${currentWidth}px`,
                height: `${thumbnailRef.offsetHeight}px`,
                overflow: 'visible'
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
    <p>텍스트를 클릭하여 텍스트를 수정할 수 있습니다.</p>
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
            style="
                font-family: {selectedFontFamily};
                color: {thumbnailTextColor};
            "
        >
        </div>
        {#if isDraggingOver}
            <div class="drag-overlay">이미지를 여기에 놓으세요</div>
        {/if}
    </div>
    <br>

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
    <br>

    <!-- 정렬 컨트롤 -->
    <div class="position-control">
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
    </div>
    <br>

    <!-- JSON 저장 및 불러오기 버튼 컨트롤 추가 -->
    <div class="json-controls">
        <button type="button" class="json-btn export-btn" onclick={exportJson}>
            💾 설정 JSON 저장
        </button>

        <input 
            type="file" 
            accept=".json" 
            bind:this={jsonFileInputRef} 
            onchange={importJson} 
            id="json-file-input" 
            class="hidden-file-input" />

        <label for="json-file-input" class="json-btn import-btn">
            📂 설정 JSON 불러오기
        </label>
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
/* ================================
   JSON 컨트롤 스타일 추가
   ================================ */
.json-controls {
    display: flex;
    gap: 8px;
    margin-bottom: 8px;
}
.json-btn {
    flex: 1;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.75rem;
    background: var(--box-bg-color);
    color: var(--text-color);
    border: var(--border);
    border-radius: var(--border-radius);
    font-family: inherit;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    text-align: center;
}
.json-btn:hover {
    background: var(--button-bg-color);
}

:global(*) {
    box-sizing: border-box;
}
/* ================================
   Header
   ================================ */
header {
    max-width: 820px;
    margin: 0 auto;
    padding: 3rem 1.5rem 1.5rem;
}
header h1 {
    margin: 0;
    letter-spacing: -0.04em;
}
header p {
    margin-top: 0.75rem;
    color: var(--gray);
    line-height: 1.7;
}
/* ================================
   Main
   ================================ */
main {
    width: min(100%, 820px);
    padding: 0 1.5rem 3rem;
}
/* ================================
   Toolbar
   ================================ */
.toolbar {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 6px;
    width: 100%;
    padding: 8px;
    background: var(--box-bg-color);
    border: var(--border);
    border-radius: var(--border-radius);
}
.toolbar button,
.toolbar select,
.custom-font-input {
    height: 34px;
    margin: 0;
    font-family: inherit;
    font-size: 0.875rem;
}
.toolbar button {
    min-width: 34px;
    padding: 0 0.7rem;
}
.toolbar button.active {
    background: var(--bg-color);
    border-color: rgba(0, 0, 0, 0.3);
}
.toolbar select {
    width: auto;
    padding: 0 2rem 0 0.7rem;
}
.font-select {
    max-width: 165px;
}
.font-size-select {
    max-width: 125px;
}
.custom-font-input {
    width: 145px !important;
    padding: 0 0.7rem !important;
    margin-top: 0 !important;
}
/* ================================
   Color Picker
   ================================ */
.color-picker-label {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 4px;
    width: 42px;
    height: 34px;
    padding: 0;
    background: var(--button-bg-color);
    border: var(--border);
    border-radius: var(--border-radius);
    cursor: pointer;
}
.color-picker-label input[type="color"] {
    width: 18px;
    height: 18px;
    padding: 0;
    margin: 0;
    border: 0;
    border-radius: 50%;
    background: transparent;
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
}
.color-picker-label input[type="color"]::-webkit-color-swatch-wrapper {
    padding: 0;
}
.color-picker-label input[type="color"]::-webkit-color-swatch {
    border: 1px solid rgba(0, 0, 0, 0.2);
    border-radius: 50%;
}
.clear-btn {
    margin-left: auto !important;
}
/* ================================
   Thumbnail
   ================================ */
#thumbnail {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    margin-top: 12px;
    padding: 2em;
    display: grid;
    place-items: center;
    place-content: center;
    overflow: hidden;
    border: 2px solid var(--text-color);
    border-radius: var(--border-radius);
    background-color: #fff;
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
}
#thumbnail.drag-over {
    border-style: dashed;
}
.drag-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(245, 245, 245, 0.85);
    color: var(--text-color);
    font-size: 1.1rem;
    font-weight: 600;
    pointer-events: none;
}
/* ================================
   Tiptap
   ================================ */
.in-thumbnail-text {
    width: max-content;
    max-width: 100%;
    height: max-content;
    color: inherit;
    font-size: 2.5rem;
}
.in-thumbnail-text :global(.ProseMirror) {
    outline: none;
    overflow: visible;
    overflow-anchor: none;
    white-space: nowrap;
}
.in-thumbnail-text :global(.ProseMirror p) {
    margin: 0;
    line-height: 1.2;
}
/* ================================
   Background Image Controls
   ================================ */
.bg-image-controls {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    width: 100%;
    margin-top: 4px;
}
.hidden-file-input {
    display: none;
}
.bg-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.55rem 0.9rem;
    background: var(--button-bg-color);
    color: var(--text-color);
    border: var(--border);
    border-radius: var(--border-radius);
    font-family: inherit;
    font-size: 0.875rem;
    font-weight: 500;
    cursor: pointer;
}
.remove-bg-btn {
    color: var(--text-color);
}
/* ================================
   Position Selector
   ================================ */
.position-control {
    width: 100%;
    margin-top: 0.5rem;
}
.position-control > p {
    margin: 0 0 0.5rem;
    color: var(--gray);
    font-size: 0.875rem;
    font-weight: 600;
}
#position-select {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: repeat(3, 1fr);
    width: 100px;
    height: 100px;
    gap: 4px;
    padding: 4px;
    background: var(--box-bg-color);
    border: var(--border);
    border-radius: var(--border-radius);
}
#position-select input[type="radio"] {
    appearance: none;
    -webkit-appearance: none;
    width: auto;
    height: auto;
    margin: 0;
    background: var(--bg-color);
    border: var(--border);
    border-radius: 4px;
    cursor: pointer;
}
#position-select input[type="radio"]:checked {
    background: var(--text-color);
    border-color: var(--text-color);
}
/* ================================
   Position
   ================================ */
#thumbnail.top-left { place-content: start start; }
#thumbnail.top-center { place-content: start center; }
#thumbnail.top-right { place-content: start end; }
#thumbnail.center-left { place-content: center start; }
#thumbnail.center-center { place-content: center center; }
#thumbnail.center-right { place-content: center end; }
#thumbnail.bottom-left { place-content: end start; }
#thumbnail.bottom-center { place-content: end center; }
#thumbnail.bottom-right { place-content: end end; }

/* ================================
   Text Alignment
   ================================ */
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

/* ================================
   Download
   ================================ */
.download-btn {
    width: 100%;
    margin-top: 12px;
    background: var(--button-bg-color);
    color: var(--text-color);
    border: var(--border);
    border-bottom: 4px solid rgba(0, 0, 0, 0.2);
    border-radius: var(--border-radius);
    font-family: inherit;
    font-size: 1rem;
    font-weight: 600;
}
.download-btn:hover {
    background: var(--box-bg-color);
}

/* ================================
   Footer
   ================================ */
footer {
    margin-top: 2rem !important;
    padding: 4rem 1.5rem !important;
    line-height: 1.8;
}

/* ================================
   Mobile
   ================================ */
@media (max-width: 600px) {
    header {
        padding-top: 2rem;
    }
    main {
        padding-left: 1rem;
        padding-right: 1rem;
    }
    .toolbar {
        gap: 5px;
    }
    .font-select,
    .font-size-select,
    .custom-font-input {
        flex: 1 1 calc(50% - 5px);
        width: auto !important;
        max-width: none;
    }
    .clear-btn {
        margin-left: 0 !important;
    }
    .color-picker-label {
        flex: 1;
    }
    #thumbnail {
        padding: 1.25rem;
    }
    .in-thumbnail-text {
        font-size: 2rem;
    }
}
</style>