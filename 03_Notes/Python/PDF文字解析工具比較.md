```python
import os
import sys
from pathlib import Path
from typing import List, Tuple
import warnings

# 隱藏 pdfminer 的警告訊息
warnings.filterwarnings('ignore')


class PDFTextExtractor:
    """
    專案用途：ESG 企業 PDF → 純文字
    功能：
    - 批次讀取 PDF
    - 多重引擎自動切換（pdfminer → pypdf → PyPDF2）
    - 自動輸出 txt
    - 保留檔名（方便後續做 labeling 與 MySQL metadata 管理）
    """

    def __init__(self, input_dir: str, output_dir: str, silent_mode: bool = True):
        self.input_dir = Path(input_dir)
        self.output_dir = Path(output_dir)
        self.silent_mode = silent_mode  # 是否隱藏 pdfminer 的 stderr 警告
        
        # 檢查輸入資料夾是否存在
        if not self.input_dir.exists():
            raise FileNotFoundError(f"❌ 輸入資料夾不存在：{self.input_dir}")
        
        # 自動建立輸出資料夾
        self.output_dir.mkdir(parents=True, exist_ok=True)
        print(f"📁 輸入資料夾：{self.input_dir}")
        print(f"📁 輸出資料夾：{self.output_dir}")

    def list_pdfs(self) -> List[Path]:
        """列出所有 PDF 檔案"""
        pdfs = list(self.input_dir.glob("*.pdf"))
        
        # 顯示找到的檔案
        if pdfs:
            print(f"\n📄 找到 {len(pdfs)} 個 PDF 檔案：")
            for pdf in pdfs:
                size_mb = pdf.stat().st_size / (1024 * 1024)
                print(f"   • {pdf.name} ({size_mb:.2f} MB)")
        
        return pdfs

    def extract_with_pdfminer(self, pdf_path: Path) -> Tuple[str, bool]:
        """方法 1: 使用 pdfminer.six（最準確，但可能失敗）"""
        try:
            from pdfminer.high_level import extract_text
            
            # 隱藏 stderr 輸出（那些 "Cannot set gray stroke color" 警告）
            if self.silent_mode:
                stderr_backup = sys.stderr
                sys.stderr = open(os.devnull, 'w')
            
            try:
                text = extract_text(pdf_path)
            finally:
                if self.silent_mode:
                    sys.stderr.close()
                    sys.stderr = stderr_backup
            
            return text, True
        except Exception as e:
            if self.silent_mode and 'sys.stderr' in locals():
                sys.stderr.close()
                sys.stderr = stderr_backup
            return f"pdfminer 失敗: {str(e)[:100]}", False

    def extract_with_pypdf(self, pdf_path: Path) -> Tuple[str, bool]:
        """方法 2: 使用 pypdf（較新版本，相容性好）"""
        try:
            from pypdf import PdfReader
            reader = PdfReader(pdf_path)
            text = ""
            for page in reader.pages:
                text += page.extract_text() + "\n"
            return text, True
        except Exception as e:
            return f"pypdf 失敗: {str(e)[:100]}", False

    def extract_with_pypdf2(self, pdf_path: Path) -> Tuple[str, bool]:
        """方法 3: 使用 PyPDF2（舊版本，作為最後備案）"""
        try:
            import PyPDF2
            text = ""
            with open(pdf_path, 'rb') as file:
                reader = PyPDF2.PdfReader(file)
                for page in reader.pages:
                    text += page.extract_text() + "\n"
            return text, True
        except Exception as e:
            return f"PyPDF2 失敗: {str(e)[:100]}", False

    def convert_pdf_to_text(self, pdf_path: Path) -> str:
        """
        將 PDF 轉成純文字
        自動嘗試多種方法，直到成功為止
        """
        print(f"\n🔄 正在解析：{pdf_path.name}...")
        
        # 依序嘗試三種方法
        extractors = [
            ("pdfminer.six", self.extract_with_pdfminer),
            ("pypdf", self.extract_with_pypdf),
            ("PyPDF2", self.extract_with_pypdf2),
        ]
        
        for engine_name, extractor in extractors:
            print(f"   🔧 嘗試使用 {engine_name}...", end=" ")
            text, success = extractor(pdf_path)
            
            if success and text.strip():
                word_count = len(text)
                print(f"✅ 成功！({word_count:,} 字元)")
                return text
            else:
                print(f"❌")
                if not success:
                    print(f"      錯誤：{text}")
        
        # 所有方法都失敗
        print(f"   ⚠️  所有解析引擎都失敗了")
        return ""

    def save_text(self, text: str, pdf_path: Path):
        """存成 .txt，檔名沿用 pdf"""
        output_path = self.output_dir / f"{pdf_path.stem}.txt"
        
        try:
            output_path.write_text(text, encoding="utf-8")
            size_kb = output_path.stat().st_size / 1024
            print(f"   💾 已儲存：{output_path.name} ({size_kb:.2f} KB)")
        except Exception as e:
            print(f"   ❌ 儲存失敗：{e}")

    def run(self):
        """Pipeline 主流程"""
        print("=" * 70)
        print("🚀 ESG PDF 轉文字工具 (多重引擎版)")
        print("=" * 70)
        
        pdfs = self.list_pdfs()

        if not pdfs:
            print("\n⚠️  找不到任何 PDF 檔案！")
            print(f"請確認以下資料夾中有 PDF 檔案：\n{self.input_dir.absolute()}")
            return

        print(f"\n開始批次處理...")
        success_count = 0
        failed_files = []
        
        for i, pdf in enumerate(pdfs, 1):
            print(f"\n{'='*70}")
            print(f"[{i}/{len(pdfs)}] 處理檔案")
            print(f"{'='*70}")
            
            text = self.convert_pdf_to_text(pdf)
            
            if text.strip():
                self.save_text(text, pdf)
                success_count += 1
            else:
                failed_files.append(pdf.name)
        
        print("\n" + "=" * 70)
        print(f"✨ 完成！成功轉換 {success_count}/{len(pdfs)} 個檔案")
        
        if failed_files:
            print(f"\n❌ 以下檔案轉換失敗：")
            for fname in failed_files:
                print(f"   • {fname}")
        
        print(f"\n📂 輸出位置：{self.output_dir.absolute()}")
        print("=" * 70)


def check_dependencies():
    """檢查必要套件是否已安裝"""
    print("🔍 檢查必要套件...")
    
    packages = {
        "pdfminer.six": "pdfminer",
        "pypdf": "pypdf",
        "PyPDF2": "PyPDF2"
    }
    
    missing = []
    for display_name, import_name in packages.items():
        try:
            __import__(import_name)
            print(f"   ✅ {display_name}")
        except ImportError:
            print(f"   ⚠️  {display_name} (未安裝，但非必須)")
            missing.append(display_name)
    
    if len(missing) == len(packages):
        print(f"\n❌ 所有 PDF 解析套件都未安裝！")
        print(f"   請至少安裝一個：pip install pdfminer.six pypdf PyPDF2")
        return False
    
    print("✅ 至少有一個套件可用\n")
    return True


if __name__ == "__main__":
    # ========== 設定區 ==========
    # 方法 1：使用原始字串（推薦）
    INPUT_DIR = r"C:\Users\TMP-214\Downloads\ESG_again"   ### 放你的 ESG 報告 PDF
    OUTPUT_DIR = r"C:\Users\TMP-214\Downloads\ESG"        ### 產生 txt 的資料夾
    
    # 方法 2：使用正斜線（也可以）
    # INPUT_DIR = "C:/Users/TMP-214/Downloads/ESG_again"
    # OUTPUT_DIR = "C:/Users/TMP-214/Downloads/ESG"
    
    # 是否隱藏 pdfminer 的警告訊息（推薦開啟）
    SILENT_MODE = True
    # ============================
    
    # 檢查套件
    if not check_dependencies():
        print("\n請先安裝至少一個 PDF 解析套件")
        print("推薦：pip install pdfminer.six")
        exit(1)
    
    try:
        extractor = PDFTextExtractor(INPUT_DIR, OUTPUT_DIR, silent_mode=SILENT_MODE)
        extractor.run()
    except FileNotFoundError as e:
        print(f"\n{e}")
        print("\n💡 請檢查：")
        print("   1. 資料夾路徑是否正確")
        print("   2. 資料夾是否存在")
        print("   3. 是否有檔案讀取權限")
    except Exception as e:
        print(f"\n❌ 發生錯誤：{e}")
        import traceback
        traceback.print_exc()