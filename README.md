# -*- coding: utf-8 -*-
"""
MedInfo-Check AI – Streamlit + OpenAI Assistants v2
(Custom Composer + OCR + Assistant-driven Compliance Analysis)
================================================================================
2025-07-17
"""

###############################################################################
# Imports & 기본 설정
###############################################################################
import os, time, tempfile, re, inspect, io, json, textwrap
from importlib.metadata import version as _pkg_version
from typing import List, Dict, Tuple, Any

import streamlit as st
import openai

import fitz, pdfplumber, docx2txt
from pptx import Presentation
from pptx.enum.shapes import MSO_SHAPE_TYPE

# OCR
try:
    from PIL import Image
except ImportError:
    Image = None
try:
    import pytesseract
except ImportError:
    pytesseract = None

###############################################################################
# Page config
###############################################################################
st.set_page_config(page_title="MedInfo-Check AI", page_icon="💊", layout="wide")

###############################################################################
# Inject CSS (ChatGPT-like 하단 고정 입력창 스타일)
###############################################################################
CUSTOM_CSS = """
<style>
.block-container { padding-bottom: 8rem !important; } /* 하단 컴포저 공간 확보 */
#chat-composer-container {
    position: fixed; left: 0; right: 0; bottom: 0; z-index: 999;
    padding: 0.75rem 1.25rem;
    background: var(--background-color, #0e1117);
    box-shadow: 0 -2px 16px rgb(0 0 0 / 40%);
}
.chat-composer-box {
    width: 100%; max-width: 900px; margin: 0 auto;
    background: rgba(250,250,250,0.05);
    border: 1px solid rgba(250,250,250,0.15);
    border-radius: 12px;
    padding: 0.75rem 1rem;
}
.chat-composer-box .stFileUploader label { display: none !important; }
.chat-composer-box .stButton>button { width: 100%; border-radius: 8px; }
.chat-composer-box textarea { min-height: 80px !important; }
</style>
"""
st.markdown(CUSTOM_CSS, unsafe_allow_html=True)

###############################################################################
# OpenAI 설정 (섹션 우선, 루트 키 fallback)
###############################################################################
SEC = st.secrets.get("openai", {})
ROOT = st.secrets

OPENAI_API_KEY = SEC.get("api_key") or ROOT.get("OPENAI_API_KEY", "") or ROOT.get("api_key", "")
ASSISTANT_ID   = SEC.get("assistant_id") or ROOT.get("ASSISTANT_ID", "")
MODEL_NAME     = SEC.get("model") or ROOT.get("MODEL_NAME", "gpt-4o-mini")
OPENAI_ORG     = SEC.get("org") or ROOT.get("OPENAI_ORG", "")

if not OPENAI_API_KEY:
    st.error("OpenAI API 키가 없습니다 – .streamlit/secrets.toml 확인!")
    st.stop()

openai.api_key = OPENAI_API_KEY
if OPENAI_ORG:
    openai.organization = OPENAI_ORG

# openai 패키지 버전 확인
try:
    major, minor = map(int, _pkg_version("openai").split(".")[:2])
    assert (major, minor) >= (1, 14)
except Exception:
    st.error("openai 1.14.0 이상이 필요합니다  →  pip install -U 'openai>=1.14.0'")
    st.stop()

###############################################################################
# Assistant ID 조회 (생성 없음)
###############################################################################
def get_assistant_id() -> str:
    """secrets 에 지정된 기존 Assistant ID 반환."""
    if not ASSISTANT_ID or not ASSISTANT_ID.strip():
        st.error("기존 Assistant ID(assistant_id)가 secrets.toml 에 설정되지 않았습니다.")
        st.stop()
    return ASSISTANT_ID.strip()

###############################################################################
# 세션 상태 초기화
###############################################################################
def ensure_chat_state():
    if "chat_thread" not in st.session_state:
        th = openai.beta.threads.create()
        st.session_state.chat_thread = th.id
    if "chats" not in st.session_state:
        st.session_state.chats = []   # [(role, md_text)]
    if "active_run_id" not in st.session_state:
        st.session_state.active_run_id = None

###############################################################################
# Run 상태 폴링
###############################################################################
def _wait_for_run_completion(thread_id: str, run_id: str, poll: float = 0.3, timeout: float = 300.0):
    """주기적으로 Run 상태를 확인하고 완료되면 Run 객체 반환."""
    start = time.time()
    while True:
        run = openai.beta.threads.runs.retrieve(thread_id=thread_id, run_id=run_id)
        if run.status in {"completed", "failed", "cancelled", "expired"}:
            return run
        # file_search 등 자동 툴 사용 중: in_progress, queued 등 계속 대기
        if time.time() - start > timeout:
            raise TimeoutError("Run 처리 시간이 시간 제한을 초과했습니다.")
        time.sleep(poll)

###############################################################################
# 텍스트 정리 & Bullet Split
###############################################################################
def clean_text(txt: str) -> str:
    repl = {
        "\u2011": "-",
        "\u2013": "-",
        "\u2014": "-",
        "\u2018": "'",
        "\u2019": "'",
        "\u201C": '"',
        "\u201D": '"',
        "\u2022": "*",
    }
    for bad, good in repl.items():
        txt = txt.replace(bad, good)
    return " ".join(txt.split())

def split_bullets(s: str) -> List[str]:
    return [ln.lstrip("*-• ").strip() for ln in s.splitlines() if ln.strip()]

###############################################################################
# File → 텍스트 변환 (로컬 요약/위험 탐지용)
###############################################################################
def extract_text(file) -> str:
    """UploadedFile 을 받아 텍스트 추출."""
    ext = os.path.splitext(file.name)[1].lower()
    file.seek(0)
    try:
        if ext == ".pdf":
            with fitz.open(stream=file.read(), filetype="pdf") as doc:
                return clean_text("\n".join(p.get_text() for p in doc))
        if ext in {".doc", ".docx"}:
            data = file.read()
            tmp = tempfile.mktemp(suffix=ext)
            with open(tmp, "wb") as f:
                f.write(data)
            txt = docx2txt.process(tmp)
            os.remove(tmp)
            return clean_text(txt)
        if ext in {".ppt", ".pptx"}:
            data = file.read()
            tmp = tempfile.mktemp(suffix=ext)
            with open(tmp, "wb") as f:
                f.write(data)
            prs = Presentation(tmp)
            txts = []
            for slide in prs.slides:
                for shp in slide.shapes:
                    if shp.shape_type == MSO_SHAPE_TYPE.TEXT_BOX or getattr(shp, "text", None):
                        txts.append(shp.text)
            os.remove(tmp)
            return clean_text("\n".join(txts))
    except Exception:
        pass
    # fallback
    if ext == ".pdf":
        file.seek(0)
        with pdfplumber.open(file) as pdf:
            return clean_text("\n".join(p.extract_text() or "" for p in pdf.pages))
    return ""

###############################################################################
# 간단 규칙 기반 위험 탐지 (즉시 경고용)
###############################################################################
DEFAULT_RULES = {
    "banned_keywords": ["100% 치료", "완치", "부작용 없음"],
    "off_label_patterns": [r"암\s*치료", r"체중\s*감소"],
}
def detect_risk(text: str) -> List[Tuple[str, str]]:
    result = [(kw, "banned_keyword") for kw in DEFAULT_RULES["banned_keywords"] if kw in text]
    for pat in DEFAULT_RULES["off_label_patterns"]:
        result += [(m.group(0), "off_label") for m in re.finditer(pat, text, flags=re.I)]
    return result

###############################################################################
# 내장 규정 요약 (규정 문서 없을 때 사용)
###############################################################################
BUILTIN_REG_SUMMARY = textwrap.dedent("""
주요 한국 의약품광고 심의 체크포인트 (요약):
- 허가/신고사항 외 효능·효과, 용법·용량 광고 금지.
- 사실과 다르거나 소비자를 오인하게 하는 표현 금지.
- '최고', '최상', '100%', '완치', '부작용 없음' 등 절대적/과장 표현 금지.
- 부작용을 은폐하거나 안전성을 과장하는 표현 금지.
- 특정 전문가(의사/약사 등) 명의 추천 광고 금지(법령 허용 예외 제외).
- 사용 전후 비교, 체험담/감사장/주문 쇄도 등 증언성 광고 주의.
""").strip()

###############################################################################
# OCR 유틸
###############################################################################
def ocr_image_from_bytes(data: bytes, lang: str = "eng+kor") -> str:
    if Image is None or pytesseract is None:
        return ""
    try:
        img = Image.open(io.BytesIO(data)).convert("RGB")
        return pytesseract.image_to_string(img, lang=lang)
    except Exception:
        return ""

###############################################################################
# 파일 업로드 → OpenAI Files (공통)
###############################################################################
IMAGE_EXTS = {".png", ".jpg", ".jpeg", ".gif", ".webp"}
DOC_EXTS   = {".pdf", ".doc", ".docx", ".ppt", ".pptx", ".txt", ".md", ".html", ".htm"}

# OpenAI File Search 지원 확장자(안전 서브셋)
SUPPORTED_FILE_SEARCH_EXTS = {
    ".c",".cs",".cpp",".doc",".docx",".html",".htm",".java",".json",".md",".pdf",
    ".php",".pptx",".py",".rb",".tex",".txt",".css",".js",".sh",".ts",
}

def _upload_single_file_bytes(data: bytes, original_name: str, purpose: str):
    ext = os.path.splitext(original_name)[1] or ".bin"
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix=ext)
    try:
        tmp.write(data)
        tmp.flush(); tmp.close()
        with open(tmp.name, "rb") as fh:
            resp = openai.files.create(file=fh, purpose=purpose)
    finally:
        try: os.remove(tmp.name)
        except Exception: pass
    return resp

def upload_files_generic(files, purpose_map=None, force_purpose=None, ocr=False):
    """공통 업로드: Streamlit UploadedFile 리스트 → OpenAI Files + 메타."""
    results=[]
    for uf in files:
        uf.seek(0); data = uf.read()
        ext = os.path.splitext(uf.name)[1].lower()
        if force_purpose:
            purpose = force_purpose
        else:
            if purpose_map is not None:
                purpose = purpose_map(ext)
            else:
                purpose = "vision" if ext in IMAGE_EXTS else "assistants"
        resp = _upload_single_file_bytes(data, uf.name, purpose)
        item = {
            "file_id": resp.id,
            "name": uf.name,
            "ext": ext,
            "is_image": ext in IMAGE_EXTS,
            "retrieval_ok": (ext in SUPPORTED_FILE_SEARCH_EXTS),
        }
        if ocr and item["is_image"]:
            item["ocr_text"] = ocr_image_from_bytes(data)
        results.append(item)
    return results

###############################################################################
# Assistant 분석 프롬프트 생성
###############################################################################
def build_compliance_prompt(filenames_review: List[str],
                            filenames_refs: List[str],
                            filenames_regs: List[str],
                            reg_summary: str) -> str:
    return f"""
당신은 한국제약바이오협회 의약품광고심의위원회 심의관입니다.

아래 첨부된 **검토 자료(Review Materials)** 에 포함된 모든 광고/프로모션성 주장(Key Messages)을 추출하고,
첨부된 **규정 문서(Regulations)** 및 **참고 문헌(References)** 을 근거로
한국 의약품광고 관련 법령 및 심의기준 준수 여부를 평가하세요.

### 파일 분류
- Review: {', '.join(filenames_review) or '(없음)'}
- References: {', '.join(filenames_refs) or '(없음)'}
- Regulations: {', '.join(filenames_regs) or '(없음)'}

### 규정 요약
{reg_summary}

### 평가 기준 태그
허가외효능, 과장, 절대표현, 안전성과장, 전문가추천, 비교표시, 소비자오인, 기타

### 출력 형식(JSON만):
[
 {{"claim":"...", "status":"COMPLIANT|MINOR_FIX|POTENTIAL_VIOLATION|VIOLATION|NEEDS_EVIDENCE",
   "issue_tags":["허가외효능",...],
   "law_refs":["약사법 제68조","규칙 별표7 2.가"],
   "rationale_kor":"왜 해당 등급인지 간략 설명",
   "suggestion_kor":"규정 부합하도록 수정안"
 }},
 ...
]

JSON 이외 텍스트는 출력하지 마세요.
""".strip()

###############################################################################
# Assistant 분석 실행
###############################################################################
def run_assistant_compliance_analysis(rv_meta: List[Dict],
                                      ref_meta: List[Dict],
                                      reg_meta: List[Dict],
                                      img_meta: List[Dict],
                                      reg_text_fallback: str) -> Dict:
    """
    첨부 메타를 기반으로 Assistant에게 규정 준수 분석을 요청하고 결과(JSON)를 반환.
    rv_meta/ref_meta/reg_meta/img_meta: upload_files_generic() 결과 리스트.
    """
    # 새 Thread (분석 전용)
    thread = openai.beta.threads.create()
    thread_id = thread.id

    # 파일 검색 첨부: 지원 확장자만
    all_docs = rv_meta + ref_meta + reg_meta
    attachments = [
        {"file_id": m["file_id"], "tools": [{"type": "file_search"}]}
        for m in all_docs if m.get("retrieval_ok")
    ]

    # 규정 요약 텍스트
    filenames_review = [m["name"] for m in rv_meta]
    filenames_refs   = [m["name"] for m in ref_meta]
    filenames_regs   = [m["name"] for m in reg_meta]

    reg_summary = reg_text_fallback or BUILTIN_REG_SUMMARY

    prompt_text = build_compliance_prompt(filenames_review, filenames_refs, filenames_regs, reg_summary)

    # 메시지 content: 분석 프롬프트 텍스트 + 업로드 이미지(검토 이미지)
    content = [{"type": "text", "text": prompt_text}]
    for m in img_meta:
        content.append({"type": "image_file", "image_file": {"file_id": m["file_id"]}})
        if m.get("ocr_text"):
            content.append({"type": "text", "text": f"[OCR {m['name']}]\n{m['ocr_text'][:2000]}"} )

    # 메시지 생성
    openai.beta.threads.messages.create(
        thread_id=thread_id,
        role="user",
        content=content,
        attachments=attachments if attachments else None,
    )

    # Run 생성 (추가 instructions 없이; Assistant 기본 + 메시지 프롬프트 기반)
    run = openai.beta.threads.runs.create(
        thread_id=thread_id,
        assistant_id=get_assistant_id(),
    )

    # 대기
    with st.spinner("규정 준수 분석 중…(Assistant)"):
        run = _wait_for_run_completion(thread_id, run.id)

    if run.status != "completed":
        return {"error": f"run status={run.status}"}

    # 메시지 수집
    msgs = openai.beta.threads.messages.list(thread_id=thread_id, order="desc")

    raw = None
    for m in msgs.data:
        if m.role == "assistant":
            parts = []
            for part in getattr(m, "content", []):
                t = getattr(part, "type", None)
                if t == "text" and hasattr(part, "text"):
                    parts.append(part.text.value)
                elif isinstance(part, dict) and part.get("type") == "text":
                    parts.append(part.get("text", ""))
            raw = "\n".join(parts).strip()
            break

    if raw is None:
        return {"error": "no assistant response"}

    # JSON 파싱 시도
    parsed = None
    try:
        parsed = json.loads(raw)
    except Exception:
        # JSON 추출 (첫 [ ... ] 범위)
        m = re.search(r'(\[.*\])', raw, flags=re.S)
        if m:
            try:
                parsed = json.loads(m.group(1))
            except Exception:
                pass
    if not isinstance(parsed, list):
        parsed = []

    return {
        "assistant_raw": raw,
        "parsed": parsed,
    }

###############################################################################
# Chat 제출 처리 (전역 커스텀 입력창 → Chat Thread)
###############################################################################
def _send_user_message_and_run(text: str, uploaded: List[Dict]):
    """실제 API 호출 & 응답 처리 (Chat Thread)."""
    ensure_chat_state()

    # 메시지 콘텐츠 준비
    content = []
    if text:
        content.append({"type": "text", "text": text})
    for item in uploaded:
        if item["is_image"]:
            content.append({"type": "image_file", "image_file": {"file_id": item["file_id"]}})
            if item.get("ocr_text"):
                content.append({"type": "text", "text": f"[OCR {item['name']}]\n{item['ocr_text'][:2000]}"} )

    # 문서 첨부 (검색/참조용) — 지원 확장자만
    doc_attachments = [
        {"file_id": item["file_id"], "tools": [{"type": "file_search"}]}
        for item in uploaded if (not item["is_image"]) and item.get("retrieval_ok")
    ]
    skipped_docs = [item["name"] for item in uploaded if (not item["is_image"]) and not item.get("retrieval_ok")]

    if skipped_docs:
        content.append({
            "type": "text",
            "text": f"(file_search 미지원 확장자: {', '.join(skipped_docs)})"
        })

    if not content:
        content.append({"type": "text", "text": "(파일 첨부)"})

    # 메시지 생성
    try:
        openai.beta.threads.messages.create(
            thread_id=st.session_state.chat_thread,
            role="user",
            content=content,
            attachments=doc_attachments if doc_attachments else None,
        )
    except Exception as e:
        st.error(f"메시지 생성 실패: {e}")
        return

    # 로컬 히스토리 표시
    attach_names = ", ".join(it["name"] for it in uploaded) if uploaded else ""
    display_md = text or ""
    if attach_names:
        display_md += f"\n\n📎 첨부: {attach_names}"
        if skipped_docs:
            display_md += f"\n⚠️ 검색 미지원 확장자: {', '.join(skipped_docs)}"
    st.session_state.chats.append(("user", display_md))
    st.chat_message("user").markdown(display_md)

    # Run 생성
    try:
        run = openai.beta.threads.runs.create(
            thread_id=st.session_state.chat_thread,
            assistant_id=get_assistant_id(),
        )
    except Exception as e:
        st.error(f"Run 생성 실패: {e}")
        return
    st.session_state.active_run_id = run.id

    # 대기
    with st.spinner("Assistant typing…"):
        try:
            run = _wait_for_run_completion(st.session_state.chat_thread, run.id)
        except TimeoutError as e:
            st.error(str(e)); return
    st.session_state.active_run_id = None

    if run.status != "completed":
        st.error(f"Assistant run failed: {run.status}")
        return

    # 응답
    msgs = openai.beta.threads.messages.list(thread_id=st.session_state.chat_thread, order="desc")
    ans = "(no response)"
    for m in msgs.data:
        if m.role == "assistant":
            parts=[]
            for part in getattr(m,"content",[]):
                if getattr(part,"type",None)=="text" and hasattr(part,"text"):
                    parts.append(part.text.value)
                elif isinstance(part,dict) and part.get("type")=="text":
                    parts.append(part.get("text",""))
            if parts: ans="\n".join(parts)
            break

    st.session_state.chats.append(("assistant", ans))
    st.chat_message("assistant").markdown(ans)

def process_chat_submission_from_form(text: str, files: List[Any]):
    """커스텀 폼 제출값(text, files)을 Assistants Thread 로 전송."""
    if not text and not files:
        return
    active_id = st.session_state.get("active_run_id")
    if active_id:
        _wait_for_run_completion(st.session_state.chat_thread, active_id)
        st.session_state.active_run_id = None
    uploaded = upload_files_generic(files, ocr=True)
    _send_user_message_and_run(text, uploaded)

###############################################################################
# Chat 섹션 (히스토리 표시만)
###############################################################################
def chat_ui():
    """Chat 섹션에 대화 히스토리 표시 (입력창은 하단 커스텀)."""
    ensure_chat_state()
    st.subheader("💬 Chat with AI")
    for role, msg in st.session_state.chats:
        st.chat_message(role).markdown(msg)
    st.caption("아래 입력창에서 메시지를 보내세요. 문서/이미지 첨부 지원.")

###############################################################################
# Document Analysis 섹션 (Assistant 호출)
###############################################################################
def analysis_ui():
    st.subheader("📑 Document Analysis")

    rv_files  = st.file_uploader("검토 자료", ["pdf", "doc", "docx", "ppt", "pptx"], accept_multiple_files=True)
    ref_files = st.file_uploader("참고 논문", ["pdf", "doc", "docx", "ppt", "pptx"], accept_multiple_files=True)
    reg_files = st.file_uploader("규정 문서 (선택)", ["pdf", "doc", "docx", "ppt", "pptx"], accept_multiple_files=True)
    img_files = st.file_uploader("광고 이미지 (선택)", ["png", "jpg", "jpeg", "gif", "webp"], accept_multiple_files=True)

    if rv_files and st.button("🚀 분석 시작"):
        # 로컬 텍스트 추출 → 위험 패턴
        with st.spinner("로컬 텍스트 추출 중…"):
            tgt_text  = "\n\n".join(extract_text(f) for f in rv_files)
            ref_texts = [extract_text(f) for f in (ref_files or [])]
            reg_texts = [extract_text(f) for f in (reg_files or [])]
            reg_text  = "\n\n".join(rt for rt in reg_texts if rt.strip())

        risk = detect_risk(tgt_text)

        # 파일 업로드(OpenAI) - 분석용
        with st.spinner("파일 업로드 중…"):
            rv_meta  = upload_files_generic(rv_files, ocr=False)   # 문서
            ref_meta = upload_files_generic(ref_files or [], ocr=False)
            reg_meta = upload_files_generic(reg_files or [], ocr=False)
            img_meta = upload_files_generic(img_files or [], ocr=True)  # 이미지+OCR

        # Assistant 분석 요청
        res_assistant = run_assistant_compliance_analysis(
            rv_meta=rv_meta,
            ref_meta=ref_meta,
            reg_meta=reg_meta,
            img_meta=img_meta,
            reg_text_fallback=reg_text or BUILTIN_REG_SUMMARY,
        )

        st.success("분석 완료!")

        # --- 위험 표현 (로컬 룰) ------------------------------------------------
        with st.expander("⚠️ 위험 표현 (간이 탐지)"):
            if not risk:
                st.write("위험 표현이 발견되지 않았습니다.")
            else:
                for p, t in risk:
                    st.write(f"• {p} | {t}")

        # --- Assistant 규정 준수 분석 ------------------------------------------
        with st.expander("✅ 규정 준수 (Assistant 분석)"):
            if "error" in res_assistant:
                st.error(res_assistant["error"])
            else:
                parsed = res_assistant.get("parsed", [])
                if not parsed:
                    st.write("JSON 파싱 실패. 원문:")
                    st.code(res_assistant.get("assistant_raw","(없음)"), language="json")
                else:
                    for item in parsed:
                        icon = {
                            "COMPLIANT": "✅",
                            "MINOR_FIX": "🟡",
                            "POTENTIAL_VIOLATION": "⚠️",
                            "VIOLATION": "🚫",
                            "NEEDS_EVIDENCE": "❓",
                        }.get(item.get("status",""), "❓")
                        st.markdown(f"**{icon} {item.get('claim','')}**")
                        if item.get("rationale_kor"): st.write(item["rationale_kor"])
                        if item.get("law_refs"):
                            st.caption("근거: " + ", ".join(item["law_refs"]))
                        if item.get("suggestion_kor"):
                            st.info(item["suggestion_kor"])
                        st.markdown("---")

        # --- 원문 JSON 보기 -----------------------------------------------------
        with st.expander("🧾 Assistant Raw JSON 응답"):
            st.code(res_assistant.get("assistant_raw",""), language="json")

###############################################################################
# 하단 ChatGPT 스타일 컴포저
###############################################################################
def chat_composer(show: bool):
    """하단 고정 입력창. show=True 인 경우에만 렌더."""
    if not show:
        return

    disabled = st.session_state.get("active_run_id") is not None

    st.markdown('<div id="chat-composer-container">', unsafe_allow_html=True)
    with st.container():
        st.markdown('<div class="chat-composer-box">', unsafe_allow_html=True)

        # 폼 (엔터 대신 명시적 전송 버튼; clear_on_submit=True)
        with st.form("chat_composer_form", clear_on_submit=True):
            files = st.file_uploader(
                "파일 첨부",
                type=None,  # 모든 확장자 허용
                accept_multiple_files=True,
                disabled=disabled,
                label_visibility="collapsed",
                key="composer_files",
            )
            text = st.text_area(
                "메시지",
                placeholder="메시지를 입력하거나 위 영역에 파일을 드래그하세요…",
                disabled=disabled,
                label_visibility="collapsed",
                key="composer_text",
            )
            send_col1, send_col2, send_col3 = st.columns([6,1,1])
            with send_col2:
                submitted = st.form_submit_button("보내기", disabled=disabled)
            with send_col3:
                st.form_submit_button("취소", disabled=disabled)

        st.markdown("</div>", unsafe_allow_html=True)
    st.markdown("</div>", unsafe_allow_html=True)

    if 'submitted' in locals() and submitted and not disabled:
        process_chat_submission_from_form(text, files or [])

###############################################################################
# 메인 레이아웃
###############################################################################
def main():
    ensure_chat_state()

    st.title("MedInfo-Check AI")

    # 섹션 선택
    section = st.radio(
        " ",
        options=("📑 Document Analysis", "💬 Chat with AI"),
        index=0,
        horizontal=True,
        label_visibility="collapsed",
        key="main_section_selector",
    )

    if section.startswith("📑"):
        analysis_ui()
        show_chat_input = False
    else:
        chat_ui()
        show_chat_input = True

    # 하단 컴포저 (Chat 섹션에서만)
    chat_composer(show_chat_input)

    st.caption("© 2025 MedInfo-Check AI")

if __name__ == "__main__":
    main()
