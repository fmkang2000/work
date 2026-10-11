# VS Code 복사·붙여넣기(클립보드) 구조 분석

- 분석 대상: microsoft/vscode 공식 소스, `main` 브랜치 커밋 `48a76ca` (package.json 버전 1.142.0, 2026-10-11)
- 분석 범위: 에디터 복사/잘라내기/붙여넣기 명령, DOM 클립보드 이벤트, 붙여넣기 공급자(paste provider), 클립보드 서비스(데스크톱/웹), 메인 프로세스(Electron), 탐색기 파일 복사, 확장 API
- 실제 설치본(`C:\Users\SAMSUNG\AppData\Local\Programs\Microsoft VS Code\resources\app`)은 이 원본을 번들·압축(minify)한 결과물(`out\...\*.js`)입니다. 설치본에서 확인하는 방법은 맨 끝 "설치본 대조 방법"에 정리했습니다.

---

## 1. 결론 요약

VS Code의 클립보드는 **4개 층**으로 나뉩니다.

| 층 | 역할 | 핵심 파일 |
|---|---|---|
| ① 명령 층 | Ctrl+C/X/V, 메뉴, 명령 팔레트를 "명령"으로 등록하고 포커스 위치에 따라 구현을 고름 | `src/vs/editor/contrib/clipboard/browser/clipboard.ts` |
| ② 브라우저 이벤트 층 | 숨겨진 입력 요소(textarea 또는 EditContext)에서 DOM `copy`/`cut`/`paste` 이벤트를 받아 텍스트+메타데이터를 클립보드에 기록/해석 | `src/vs/editor/browser/controller/editContext/clipboardUtils.ts`, `.../native/nativeEditContext.ts`, `.../textArea/textAreaEditContextInput.ts` |
| ③ 붙여넣기 확장 층 | 붙여넣을 데이터를 언어별·확장별 "붙여넣기 공급자"에 넘겨 다른 형태(경로, 링크, 이미지 등)로 변환할지 결정 | `src/vs/editor/contrib/dropOrPasteInto/browser/copyPasteController.ts`, `defaultProviders.ts` |
| ④ 시스템 클립보드 서비스 층 | OS 클립보드 읽기/쓰기. 데스크톱은 IPC → 메인 프로세스 → Electron `clipboard` 모듈, 웹은 `navigator.clipboard` | `src/vs/platform/clipboard/common/clipboardService.ts`, `src/vs/workbench/services/clipboard/electron-browser/clipboardService.ts`, `src/vs/platform/native/electron-main/nativeHostMainService.ts`, `src/vs/platform/clipboard/browser/clipboardService.ts` |

핵심 설계 특징 3가지:
1. **복사는 브라우저 표준 이벤트에 얹혀 동작**: 명령이 `document.execCommand('copy')`를 호출 → DOM `copy` 이벤트 발생 → 그 핸들러가 클립보드에 기록.
2. **붙여넣기는 Electron에게 "진짜 붙여넣기"를 시킴**: 렌더러가 메인 프로세스에 `webContents.paste()`를 요청 → DOM `paste` 이벤트 발생 → 그 핸들러가 처리.
3. **텍스트 외에 "VS Code 전용 메타데이터"를 함께 실음**: 줄 단위 복사인지, 다중 커서 복사인지, 열(블록) 선택인지 등을 별도 MIME 형식으로 같이 저장하여 붙여넣기 때 원래 의도대로 복원.

---

## 2. 복사(Ctrl+C) 흐름

```
[Ctrl+C / 메뉴 "Copy"]
   │  명령 ID: editor.action.clipboardCopyAction (MultiCommand)
   ▼
① clipboard.ts  ─ 우선순위 10000 'code-editor' 구현: 에디터에 텍스트 포커스가 있으면
   │               executeClipboardCopyWithWorkaround()
   │                 └ document.execCommand('copy')
   │               (포커스가 다른 곳이면 우선순위 0 'generic-dom' 구현: activeDocument.execCommand('copy'))
   ▼
② DOM 'copy' 이벤트 → nativeEditContext.ts / textAreaEditContextInput.ts
   │   CopyOptions.electronBugWorkaroundCopyEventHasFired = true
   │   createClipboardCopyEvent()  ← 선택 영역에서 text/html/메타데이터 생성
   │   onWillCopy 이벤트 발행 ──► ③ CopyPasteController.handleCopy() (공급자에게 복사 시점 데이터 준비 요청)
   │   ensureClipboardGetsEditorData()
   │       e.preventDefault()
   │       clipboardData.setData('text/plain', 텍스트)
   │       clipboardData.setData('text/html', 구문강조 HTML)   ← 옵션 켜진 경우
   │       clipboardData.setData('vscode-editor-data', JSON 메타데이터)
   │       InMemoryClipboardMetadataManager에 (텍스트, 메타데이터) 기억
   ▼
[OS 클립보드]
```

근거:
- 단축키 등록: `clipboard.ts:78-111` — `KeyMod.CtrlCmd | KeyCode.KeyC`, Windows 보조키 `Ctrl+Insert`. 브라우저(웹판)에서는 단축키를 등록하지 않음(`platform.isNative ? {...} : undefined`). 주석: "browsers do that for us and it avoids security prompts".
- 포커스별 분기: `clipboard.ts:211-262` `registerExecCommandImpl()` — 에디터(우선순위 10000) → 일반 DOM(우선순위 0).
- 빈 선택 복사: `clipboard.ts:225-229` — 선택이 없고 `editor.emptySelectionClipboard`가 꺼져 있으면 아무것도 하지 않음. 켜져 있으면(기본값) 커서가 있는 **줄 전체**가 복사됨.
- 데이터 생성: `clipboardUtils.ts:47-75` `getDataToCopy()` — `isBlock`(열 선택), `isFromEmptySelection`(줄 복사), `multicursorText`(커서별 텍스트 배열), `text`, `html`, `mode`(언어 ID).
- 구문 강조 HTML: `clipboardUtils.ts:58` — `editor.copyWithSyntaxHighlighting`이 켜져 있고 **65,536자 미만**일 때만 HTML 생성(대용량 복사 성능 보호).
- 클립보드 기록: `clipboardUtils.ts:158-164` `setTextData()` — `text/plain`, `text/html`, `vscode-editor-data` 3종 기록.

### Electron 버그 우회 장치
`clipboard.ts:194-209`
```ts
CopyOptions.electronBugWorkaroundCopyEventHasFired = false;
editor.getContainerDomNode().ownerDocument.execCommand('copy');
if (platform.isNative && CopyOptions.electronBugWorkaroundCopyEventHasFired === false) {
    // copy 이벤트가 안 왔다 → Electron 버그로 판단
    const { dataToCopy } = generateDataToCopyAndStoreInMemory(...);
    clipboardService.writeText(dataToCopy.text);   // 텍스트만 직접 기록
}
```
- `execCommand('copy')` 호출 후 DOM `copy` 이벤트가 오지 않으면(코드 주석상 "Electron 버그로 추정") 클립보드 서비스로 **일반 텍스트만** 직접 씁니다. 이 경우 HTML·메타데이터는 OS 클립보드에 실리지 않고, 메모리 메타데이터만 남습니다.

---

## 3. 잘라내기(Ctrl+X) 흐름

복사와 같되, DOM `cut` 이벤트 핸들러가 클립보드 기록 후 실제 삭제를 수행합니다.
- `nativeEditContext.ts` `cut` 핸들러: `ensureClipboardGetsEditorData()` → `this._viewController.cut()`.
- 예외: EditContext 모드에서는 `execCommand('cut')`이 동작하지 않아 **copy를 실행한 뒤 `Handler.Cut`을 직접 트리거**(`clipboard.ts:230-237`, 주석 "execCommand(copy) works for edit context, but not execCommand(cut)").
- Windows 보조키 `Shift+Delete`(`clipboard.ts:49`).

---

## 4. 붙여넣기(Ctrl+V) 흐름

```
[Ctrl+V / 메뉴 "Paste"]
   │  명령 ID: editor.action.clipboardPasteAction
   ▼
① clipboard.ts 'code-editor' 구현
   │   clipboardService.triggerPaste(windowId)
   ▼
④ NativeClipboardService.triggerPaste() ── IPC('nativeHost' 채널) ──►
   메인 프로세스 NativeHostMainService.triggerPaste()
       window.win.webContents.paste()          ← Electron이 OS 수준 붙여넣기 실행
   ▼
② DOM 'paste' 이벤트 → nativeEditContext.ts / textAreaEditContextInput.ts
   │   createClipboardPasteEvent()
   │     text = clipboardData.getData('text/plain')
   │     metadata = JSON.parse(getData('vscode-editor-data'))
   │                ?? InMemoryClipboardMetadataManager.get(text)
   │     (텍스트가 비고 파일만 있으면 파일 이름 목록을 텍스트로 사용)
   │   onWillPaste 발행 ──► ③ CopyPasteController.handlePaste()
   │        └ 해당 MIME을 처리할 공급자가 있으면 e.setHandled() 후 직접 처리
   │   (처리되지 않았으면) viewController.paste(text, pasteOnNewLine, multicursorText, mode, isBlock)
   ▼
에디터 모델: cursor.ts paste() → PasteOperation.getEdits()  (cursorTypeEditOperations.ts:702~)
   ├ 열 선택 복사본 + 커서 1개 + multiCursorPaste='spread' → _blockPaste (열 단위로 붙임)
   ├ 커서 수 = 복사된 커서 수(multicursorText) → _distributedPaste (커서마다 한 조각씩)
   ├ 'spread'이고 줄 수 = 커서 수 → 줄마다 분배
   ├ 줄 복사본(pasteOnNewLine) + 선택 없음 → 현재 줄 **위에** 줄 전체 삽입
   └ 그 외 → _simplePaste (선택 영역 치환)
   ※ undo 스택: shouldPushStackElementBefore/After = true → 붙여넣기 1회 = 실행 취소 1단계
```

근거:
- `clipboard.ts:277-339` — `triggerPaste()` 결과가 있으면 그것을 기다린 뒤 `CopyPasteController.finishedPaste()`까지 대기.
- `electron-browser/clipboardService.ts:25-28` — `nativeHostService.triggerPaste({ targetWindowId })`.
- `nativeHostMainService.ts:1068-1072` — `window?.win?.webContents.paste()`.
- `nativeEditContext.ts` `paste` 핸들러 — `e.preventDefault()` 후 브라우저 기본 삽입을 막고 VS Code가 직접 모델에 반영.
- 웹판 대체 경로: `triggerPaste()`가 `undefined`이면 `clipboardService.readText()`로 텍스트를 읽어 `Handler.Paste`를 직접 트리거(`clipboard.ts:308-335`).
- 줄 붙여넣기 판단: `cursorTypeEditOperations.ts` `_simplePaste()` — 텍스트의 **마지막 문자만 개행**이고 선택이 비어 있을 때만 "줄 위 삽입".

### 왜 메타데이터를 두 군데(클립보드 + 메모리)에 두나
- `vscode-editor-data` 사용자 지정 형식은 같은 VS Code 창/프로세스 간에는 유지되지만, 앞의 Electron 우회 경로나 웹판의 `readText()` 경로처럼 **텍스트만** 오갈 때는 사라집니다.
- 그래서 `InMemoryClipboardMetadataManager`(`clipboardUtils.ts:82-103`)가 "마지막으로 복사한 텍스트"와 메타데이터를 기억하고, 붙여넣는 텍스트가 **완전히 같을 때만** 메타데이터를 복원합니다. 다르면 즉시 폐기(`_lastState = null`) → 다른 프로그램에서 복사한 텍스트에 엉뚱한 메타데이터가 붙지 않음.
- Firefox는 `\r\n`을 `\n`으로 바꿔 붙이므로 비교용 키도 변환해 둠(`clipboardUtils.ts:22-29`).

---

## 5. 붙여넣기 확장 층 (Paste As / 붙여넣기 공급자)

`CopyPasteController`(`copyPasteController.ts`)가 에디터의 `onWillCopy`/`onWillCut`/`onWillPaste`를 구독합니다(125-127행).

**복사 시** (`handleCopy`, 165-227행)
- `prepareDocumentPaste`를 구현한 공급자들에게 복사 원본 범위를 넘겨 **비동기로** 데이터 준비를 시작.
- 클립보드에는 UUID 핸들만 `application/vnd.code.copymetadata` 형식으로 기록하고, 실제 준비 결과는 정적 변수 `_currentCopyOperation`에 보관(편집기 그룹 간 공유. 주석상 다중 창은 미지원 TODO).

**붙여넣기 시** (`handlePaste`, 229-307행)
1. 읽기 전용이거나 `editor.pasteAs.enabled`가 꺼져 있으면 관여하지 않음.
2. 클립보드 MIME 형식(+파일 형식, +복사 시 공급자 형식, +`text/uri-list`)을 처리할 공급자를 선별.
3. 있으면 `e.setHandled()`로 기본 붙여넣기를 막고 `doPasteInline()` 실행 → 공급자들의 `provideDocumentPasteEdits()` 결과를 수집.
4. 결과가 기본 텍스트 공급자 하나뿐이면 `applyDefaultPasteHandler()` → 결국 `Handler.Paste`로 일반 붙여넣기(619-636행).
5. 여러 결과가 있으면 하나를 적용하고, `editor.pasteAs.showPasteSelector`가 `afterPaste`이면 붙여넣은 자리 옆에 "붙여넣기 옵션" 위젯 표시. 우선순위는 설정 `editor.pasteAs.preferences`.
6. 복사 핸들이 일치하면(`mergeInDataFromCopy`, 558-591행) 복사 시 준비해 둔 데이터를 병합. `text/uri-list`가 없으면 `clipboardService.readResources()`로 탐색기에서 복사한 파일 목록을 보충.

기본 공급자(`defaultProviders.ts`): `DefaultTextPasteOrDropEditProvider`(text/plain), `PathProvider`·`RelativePathProvider`(uri-list → 경로/상대경로 삽입), `PasteHtmlProvider`(text/html). 확장(예: Markdown 이미지 붙여넣기)은 같은 인터페이스로 공급자를 추가합니다.

---

## 6. 시스템 클립보드 서비스 층

공통 인터페이스 `IClipboardService`(`platform/clipboard/common/clipboardService.ts`):
`triggerPaste`, `writeText`/`readText`, `readFindText`/`writeFindText`(macOS 찾기 전용 대지), `writeResources`/`readResources`/`hasResources`(파일 URI), `readImage`, `clearInternalState`.

| 구현 | 위치 | 실제 동작 |
|---|---|---|
| 데스크톱 `NativeClipboardService` | `workbench/services/clipboard/electron-browser/clipboardService.ts` | 모든 호출을 `INativeHostService`로 위임 → IPC `nativeHost` 채널(`ProxyChannel`) → 메인 프로세스 |
| 메인 프로세스 `NativeHostMainService` | `platform/native/electron-main/nativeHostMainService.ts:1059-1100` | Electron `clipboard.readText/writeText/readImage().toPNG()/writeBuffer/readBuffer/has/readFindText`, 붙여넣기는 `webContents.paste()` |
| 웹 `BrowserClipboardService` | `platform/clipboard/browser/clipboardService.ts` | `navigator.clipboard.writeText/readText/read/write`, 실패 시 숨은 textarea + `execCommand('copy')` 대체, Safari 전용 우회 로직, 파일 목록은 메모리 보관 |

데스크톱에서 렌더러(화면)는 OS 클립보드를 직접 만지지 않고, **모든 직접 접근은 메인 프로세스의 Electron API**로 모입니다(Electron 샌드박스 구조).

---

## 7. 에디터 밖의 클립보드 사용처

- **탐색기 파일 복사/잘라내기**: `explorerService.setToCopy()` → `clipboardService.writeResources()` → Electron `clipboard.writeBuffer('code/file-list', "URI\nURI...")`. 즉 OS 표준 파일 형식이 아닌 **VS Code 전용 형식**으로 저장. 잘라내기 여부는 메모리(`cutItems`)에만 보관.
  붙여넣기(`fileActions.ts:1305-1319`)는 OS에서 온 `FileList`(윈도 탐색기에서 복사한 파일)가 있으면 그것을, 없으면 `readResources()`를 사용.
- **확장 API `vscode.env.clipboard`**: 확장 호스트 → `MainThreadClipboard.$readText/$writeText`(`workbench/api/browser/mainThreadClipboard.ts`) → 같은 `IClipboardService`. 확장에는 텍스트 읽기/쓰기만 노출.
- **"구문 강조로 복사"**: `editor.action.clipboardCopyWithSyntaxHighlightingAction`(`clipboard.ts:156-192`) — 길이 제한 없이 HTML을 강제 생성(`forceCopyWithSyntaxHighlighting`).

---

## 8. 실무 관점 해석 (추론 포함)

| 현상 | 원인(근거) |
|---|---|
| 아무것도 선택 안 하고 Ctrl+C → 줄 전체 복사, Ctrl+V → 현재 줄 위에 삽입 | `emptySelectionClipboard`(기본 켜짐) + 메타데이터 `isFromEmptySelection` → `_simplePaste`의 줄 삽입 분기 |
| 다중 커서로 복사 후 같은 커서 수로 붙이면 커서별로 들어감 | `multicursorText` 메타데이터 → `_distributedPaste` |
| 다른 프로그램(메모장 등)에 붙이면 단순 텍스트, 워드/한글에는 색상 유지 | `text/plain` + `text/html`(65,536자 미만, `copyWithSyntaxHighlighting`) 동시 기록 |
| 매우 큰 코드 복사 시 색상 서식이 빠짐 | 65,536자 이상이면 HTML 생략 |
| 붙여넣은 뒤 작은 "붙여넣기 옵션" 버튼이 뜸 | `CopyPasteController`의 공급자 결과가 2개 이상 |
| 간헐적으로 서식·줄 단위 정보가 사라짐 | Electron 버그 우회 경로에서 텍스트만 기록(추정 원인, 코드 주석 근거) |

설정으로 조정 가능한 항목: `editor.emptySelectionClipboard`, `editor.copyWithSyntaxHighlighting`, `editor.multiCursorPaste`(`spread`/`full`), `editor.pasteAs.enabled`, `editor.pasteAs.showPasteSelector`, `editor.pasteAs.preferences`.

---

## 9. 설치본 대조 방법 (회장님 PC에서 확인용)

설치본 `resources\app\out` 아래는 번들된 JS라 파일 경로는 다르지만, 아래 **고유 문자열**로 같은 코드를 찾을 수 있습니다.

| 찾을 문자열 | 예상 위치(번들) | 대응 원본 |
|---|---|---|
| `vscode-editor-data` | `out\vs\workbench\workbench.desktop.main.js` | `clipboardUtils.ts` |
| `application/vnd.code.copymetadata` | 같은 파일 | `copyPasteController.ts` |
| `editor.action.clipboardPasteAction` | 같은 파일 | `clipboard.ts` |
| `code/file-list` | 같은 파일 | `electron-browser/clipboardService.ts` |
| `webContents.paste` 또는 `Triggering paste in window` | `out\main.js` | `nativeHostMainService.ts` |

PowerShell 예:
```powershell
$app = "C:\Users\SAMSUNG\AppData\Local\Programs\Microsoft VS Code\resources\app"
Get-Content "$app\package.json" | Select-String '"version"'
Select-String -Path "$app\out\vs\workbench\workbench.desktop.main.js","$app\out\main.js" `
  -Pattern 'vscode-editor-data','vnd.code.copymetadata','code/file-list','Triggering paste in window' -SimpleMatch |
  Select-Object Path, Pattern -Unique
```

[확인 필요] 설치된 VS Code 버전이 분석 기준(1.142.0 개발 브랜치)과 다르면 세부 행 번호·일부 로직(EditContext 관련 분기 등)은 다를 수 있습니다. 위 문자열이 모두 검출되면 구조는 동일하다고 볼 수 있습니다.
