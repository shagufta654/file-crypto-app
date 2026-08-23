# file-crypto-app
import streamlit as st
import os
from datetime import datetime

st.set_page_config(page_title="File Encrypt / Decrypt", page_icon="🔐", layout="wide")

st.markdown("""
    <style>
    .stButton>button {
        border-radius: 8px;
        height: 3em;
        font-weight: 600;
    }
    div[data-testid="stMetric"] {
        background-color: rgba(128,128,128,0.1);
        border-radius: 10px;
        padding: 15px;
    }
    </style>
""", unsafe_allow_html=True)

if "history" not in st.session_state:
    st.session_state.history = []


def xor_process(data: bytes, password: str) -> bytes:
    key_byte = ord(password[0])
    return bytes(byte ^ key_byte for byte in data)


def log_history(name, action_label, size):
    st.session_state.history.append({
        "Time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "File": name,
        "Action": action_label,
        "Size (bytes)": size,
    })
    # Also write to a fixed CSV file automatically, so Tableau can
    # just "Refresh" instead of needing a fresh download every time.
    import pandas as pd
    df = pd.DataFrame(st.session_state.history)
    df.to_csv("file_crypto_history_live.csv", index=False)


def show_preview(result_bytes: bytes):
    with st.expander("👁️ Preview result", expanded=True):
        try:
            text = result_bytes.decode("utf-8")
            st.text_area("", text, height=150, label_visibility="collapsed")
        except UnicodeDecodeError:
            st.warning(
                "Not readable as text — either it's scrambled (encrypted), "
                "the password was wrong, or it's a non-text file."
            )
            st.code(result_bytes[:100], language=None)


# ======================= SIDEBAR (navigation + stats only) =======================
with st.sidebar:
    st.title("🔐 File Crypto")
    page = st.radio("Navigate", ["⚡ Encrypt / Decrypt", "📜 History", "ℹ️ About"])
    st.divider()
    st.metric("Files processed this session", len(st.session_state.history))
    if st.session_state.history:
        last = st.session_state.history[-1]
        st.caption(f"Last: {last['File']} ({last['Action']}) at {last['Time']}")


# ======================= MAIN AREA =======================
if page == "⚡ Encrypt / Decrypt":
    st.header("Encrypt / Decrypt a File")
    st.caption("Lock and unlock your files with a password.")

    # ---- Stats row ----
    c1, c2, c3 = st.columns(3)
    encrypts = sum(1 for h in st.session_state.history if "Encrypt" in h["Action"])
    decrypts = sum(1 for h in st.session_state.history if "Decrypt" in h["Action"])
    c1.metric("🔒 Encrypted", encrypts)
    c2.metric("🔓 Decrypted", decrypts)
    c3.metric("📁 Total files", len(st.session_state.history))

    st.divider()

    # ---- Settings, now in main area ----
    settings_col, result_col = st.columns([1, 1.3], gap="large")

    with settings_col:
        st.subheader("Settings")
        action = st.radio("Action", ["🔒 Encrypt", "🔓 Decrypt"], horizontal=True)
        password = st.text_input("Password", type="password", placeholder="Enter password")
        method = st.radio("Source", ["📁 File path", "⬆️ Upload"], horizontal=True)

        file_path = None
        uploaded_file = None

        if method == "📁 File path":
            file_path = st.text_input("File name or path", placeholder="e.g. test.txt")
        else:
            uploaded_file = st.file_uploader("Choose a file")
            if uploaded_file is not None:
                st.caption(f"📄 {uploaded_file.name} · {len(uploaded_file.getvalue())} bytes")

        process_clicked = st.button("⚡ Process", use_container_width=True)

    with result_col:
        st.subheader("Result")

        if not password:
            st.info("Enter a password and choose a file to get started.")
        elif method == "📁 File path" and not file_path:
            st.info("Enter a file name or path.")
        elif method == "⬆️ Upload" and uploaded_file is None:
            st.info("Upload a file.")
        elif process_clicked:
            if method == "📁 File path":
                if not os.path.exists(file_path):
                    st.error(f"File not found: {file_path}")
                else:
                    with open(file_path, "rb") as f:
                        data = f.read()
                    result = xor_process(data, password)
                    with open(file_path, "wb") as f:
                        f.write(result)
                    verb = "encrypted 🔒" if action.startswith("🔒") else "decrypted 🔓"
                    st.success(f"File {verb}: **{file_path}**")
                    show_preview(result)
                    log_history(file_path, action, len(result))
            else:
                data = uploaded_file.getvalue()
                result = xor_process(data, password)
                if action.startswith("🔒"):
                    st.success("File encrypted! 🔒")
                    new_name = uploaded_file.name + ".locked"
                else:
                    st.success("File decrypted! 🔓")
                    new_name = uploaded_file.name[:-7] if uploaded_file.name.endswith(".locked") else uploaded_file.name
                show_preview(result)
                # Log the ORIGINAL name (not new_name) so Encrypt and Decrypt
                # entries for the same file always group together in charts.
                base_name = uploaded_file.name[:-7] if uploaded_file.name.endswith(".locked") else uploaded_file.name
                log_history(base_name, action, len(result))
                st.download_button(f"⬇️ Download: {new_name}", data=result, file_name=new_name, use_container_width=True)
        else:
            st.info("Fill in the settings and click **Process**.")

elif page == "📜 History":
    st.header("Processing History")
    if st.session_state.history:
        st.dataframe(st.session_state.history, use_container_width=True)

        import pandas as pd
        df = pd.DataFrame(st.session_state.history)
        csv_data = df.to_csv(index=False)

        col_a, col_b = st.columns(2)
        with col_a:
            st.download_button(
                "📊 Download CSV (for Tableau)",
                data=csv_data,
                file_name="file_crypto_history.csv",
                mime="text/csv",
                use_container_width=True,
            )
        with col_b:
            if st.button("🗑️ Clear history", use_container_width=True):
                st.session_state.history = []
                st.rerun()
    else:
        st.info("No files processed yet this session.")

elif page == "ℹ️ About":
    st.header("About this tool")
    st.write("""
    This tool uses a simple **XOR cipher** to lock and unlock files with a password.

    **How it works:** every byte of the file is combined with the first character
    of your password using an XOR operation. Running the same operation again
    with the same password reverses it — which is why the same action can both
    encrypt and decrypt.

    ⚠️ **Note:** This is built for learning purposes and is *not* secure enough
    for real sensitive data. For real security, use a proper library like
    Python's `cryptography` package (AES encryption).
    """)
