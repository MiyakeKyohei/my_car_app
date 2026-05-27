# 🚗 車種カスタマイズ & 見積もりシミュレーター (Car Customizer App)

## 📌 概要
本アプリケーションは、PythonとStreamlitを用いて構築された自動車のカスタマイズシミュレーターです。
ユーザーは好みの車種（カローラ、プリウス、クラウンなど）を選択し、ボディカラー、ホイール、内装、エンジンなどのオプションをインタラクティブに組み合わせて、リアルタイムで見積もりを作成できます。

自動車のコンセプトや価格構成をシミュレーションを通じて深く理解し、体験できるシステムとなっており、プロジェクトのデモンストレーションや研究発表のツールとしても活用できます。

## ✨ 主な機能
1. **ユーザー認証と状態管理**: アカウント作成・ログイン機能。セッション状態を保持し、ユーザーごとの体験を提供。
2. **インタラクティブなUI**: 選択肢を変更すると、即座に画面上部の合計金額やカラープレビューが更新される動的レイアウト。
3. **履歴保存機能**: 作成したカスタム状態を一時保存し、マイページからいつでも再開可能。
4. **PDF見積書の発行**: 購入手続きに進むと、現在のカスタム内容を反映したPDF資料（販売店控え）を自動生成。
5. **人気トレンドの可視化**: 過去の購入データ（CSV）を集計し、各オプションの「人気No.1」を画面にフィードバック。
6. **動的クラス拡張（拡張性）**: `__subclasses__()` を利用した動的クラス解決により、既存のコードを変更せずに新しい車種を追加可能。

## 📁 ディレクトリ構成
```text
my_car_app/
│
├── app.py                   # エントリーポイント（ログイン画面）
├── README.md                # 本ドキュメント
├── models/                  # バックエンドロジック
│   ├── __init__.py
│   ├── car.py               # 車種クラス定義（動的ファクトリ実装）
│   ├── customer.py          # 顧客情報クラス
│   └── session_manager.py   # データ永続化（CSV入出力）
├── pages/                   # Streamlitマルチページ
│   ├── 1_mypage.py          # マイページ（履歴一覧）
│   └── 2_customize.py       # カスタマイズ画面（UI・PDF生成）
└── data/                    # データ保存先（自動生成）
    ├── customers.csv        # ユーザー情報と履歴
    └── purchases.csv        # 購入実績データ
```

## 🚀 環境構築と実行方法
1. 必要なライブラリのインストール
Streamlit（Webフレームワーク）と、ReportLab（PDF生成）をインストールします。

```Bash
pip install streamlit reportlab
```
2. アプリの起動
プロジェクトのルートディレクトリ（app.pyがある場所）で以下のコマンドを実行します。

```Bash
streamlit run app.py
ブラウザが自動的に開き、http://localhost:8501 でアプリが立ち上がります。
```
## 💻 全ソースコード
以下は、本アプリケーションを構成する全てのPythonソースコードです。指定のディレクトリ構成に従って各ファイルを作成・保存してください。

### models/car.py
車両のデータ構造を定義し、動的ファクトリパターン（クラスの自動検出）を実装しています。

```Python
import json
from abc import ABC

class Car(ABC):
    # 各子クラスで共通して識別するためのクラス変数（デフォルト値）
    SERIES_NAME = ""

    def __init__(self):
        # クラス名をそのまま series に代入することも可能
        # 例: self.series = self.__class__.__name__
        self.series = self.SERIES_NAME
        self.color = ""
        self.wheel = ""
        self.interior = ""
        self.engine = ""
        
        self.series_price = 0
        self.color_price = {}
        self.wheel_price = {}
        self.interior_price = {}
        self.engine_price = {}
        self.color_codes = {}

    def get_total_price(self) -> int:
        return (self.series_price +
                self.color_price.get(self.color, 0) +
                self.wheel_price.get(self.wheel, 0) +
                self.interior_price.get(self.interior, 0) +
                self.engine_price.get(self.engine, 0))

    def get_color_code(self) -> str:
        return self.color_codes.get(self.color, "#cccccc")

    def to_string(self) -> str:
        return json.dumps({
            "series": self.series,
            "color": self.color,
            "wheel": self.wheel,
            "interior": self.interior,
            "engine": self.engine
        })

    def load_from_dict(self, data: dict):
        self.color = data.get("color", list(self.color_price.keys())[0])
        self.wheel = data.get("wheel", list(self.wheel_price.keys())[0])
        self.interior = data.get("interior", list(self.interior_price.keys())[0])
        self.engine = data.get("engine", list(self.engine_price.keys())[0])

    # === 🔥 ここがポイント：魔法の自動ファクトリ関数 ===
    @classmethod
    def create(cls, series_name: str, data_dict: dict = None):
        """
        定義されているすべての子クラスを自動探索し、
        クラス名またはSERIES_NAMEが一致するものをインスタンス化します。
        """
        for subclass in cls.__subclasses__():
            # クラス名そのもの（例: "Corolla"）か、設定された車種名（例: "カローラ"）が一致するかチェック
            if subclass.__name__ == series_name or subclass.SERIES_NAME == series_name:
                instance = subclass()
                if data_dict:
                    instance.load_from_dict(data_dict)
                return instance
        
        # 見つからなかった場合のフォールバック（デフォルトとして最初のクラスを返すか、エラーにする）
        raise ValueError(f"車種 '{series_name}' に対応するクラスが定義されていません。")

    @classmethod
    def get_all_series_names(cls) -> list:
        """現在システムに存在する（コードに書かれている）車種名を一覧で返します"""
        return [subclass.SERIES_NAME for subclass in cls.__subclasses__() if subclass.SERIES_NAME]


# --- 派生クラスの定義 (クラス名とSERIES_NAMEを定義するだけ) ---
class Corolla(Car):
    SERIES_NAME = "カローラ"
    def __init__(self):
        super().__init__()
        self.series_price = 2000000
        self.color_price = {"スーパーホワイト": 0, "プラチナホワイトパール": 33000, "ブラック": 0}
        self.wheel_price = {"15インチスチール": 0, "17インチアルミ": 55000}
        self.interior_price = {"ファブリック": 0, "合成皮革": 44000}
        self.engine_price = {"1.5L ガソリン": 0, "1.8L ハイブリッド": 350000}
        self.color_codes = {"スーパーホワイト": "#ffffff", "プラチナホワイトパール": "#f5f5f5", "ブラック": "#222222"}
        self.load_from_dict({})

class Prius(Car):
    SERIES_NAME = "プリウス"
    def __init__(self):
        super().__init__()
        self.series_price = 2750000
        self.color_price = {"アッシュ": 0, "マスタード": 0, "エモーショナルレッド": 55000}
        self.wheel_price = {"17インチアルミ": 0, "19インチアルミ": 88000}
        self.interior_price = {"ファブリック": 0, "本革": 110000}
        self.engine_price = {"1.8L ハイブリッド": 0, "2.0L ハイブリッド": 200000}
        self.color_codes = {"アッシュ": "#5a6268", "マスタード": "#e0a800", "エモーショナルレッド": "#c82333"}
        self.load_from_dict({})

class Crown(Car):
    SERIES_NAME = "クラウン"
    def __init__(self):
        super().__init__()
        self.series_price = 4350000
        self.color_price = {"プレシャスホワイト": 55000, "ブラック": 0, "ブロンズ": 55000}
        self.wheel_price = {"19インチアルミ": 0, "21インチアルミ": 121000}
        self.interior_price = {"上級ファブリック": 0, "本革": 165000}
        self.engine_price = {"2.5L ハイブリッド": 0, "2.4L ターボハイブリッド": 500000}
        self.color_codes = {"プレシャスホワイト": "#f8f9fa", "ブラック": "#222222", "ブロンズ": "#8c7853"}
        self.load_from_dict({})
```

### models/customer.py

お客様情報を保持し、JSON文字列として保存された履歴を実際のインスタンスに復元します。

```Python
import uuid
import json

class Customer:
    def __init__(self, name: str, password: str, customer_id: str = None, history: list = None):
        self.customer_id = customer_id if customer_id else str(uuid.uuid4().int)[:7]
        self.customer_name = name
        self.password = password
        self.history = history if history else []
        self.customer_car = None

    def add_history(self, car_state_str: str):
        self.history.insert(0, car_state_str)

    def get_history_instances(self):
        from models.car import Car
        instances = []
        for h_str in self.history:
            try:
                data = json.loads(h_str)
                instances.append(Car.create(data["series"], data))
            except:
                pass
        return instances
```
### models/session_manager.py
全顧客データと購入実績のCSV入出力、ログイン認証、および人気オプションの集計を行います。

```Python
import csv
import os
import atexit
from collections import Counter
from models.customer import Customer

class SessionManager:
    def __init__(self, csv_file="data/customers.csv", purchase_file="data/purchases.csv"):
        self.csv_file = csv_file
        self.purchase_file = purchase_file
        self.customer_list = []
        
        os.makedirs(os.path.dirname(self.csv_file), exist_ok=True)
        os.makedirs(os.path.dirname(self.purchase_file), exist_ok=True)
        self.load_from_csv()
        
        atexit.register(self.save_to_csv)

    def load_from_csv(self):
        if not os.path.exists(self.csv_file):
            return
        with open(self.csv_file, mode="r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                import ast
                try:
                    history = ast.literal_eval(row["history"]) if row["history"] else []
                except:
                    history = []
                customer = Customer(
                    name=row["customer_name"],
                    password=row["password"],
                    customer_id=row["customer_id"],
                    history=history
                )
                self.customer_list.append(customer)

    def save_to_csv(self):
        with open(self.csv_file, mode="w", encoding="utf-8", newline="") as f:
            fieldnames = ["customer_id", "customer_name", "password", "history"]
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            for c in self.customer_list:
                writer.writerow({
                    "customer_id": c.customer_id,
                    "customer_name": c.customer_name,
                    "password": c.password,
                    "history": str(c.history)
                })

    def login(self, name, password):
        for c in self.customer_list:
            if c.customer_name == name and c.password == password:
                return c
        return None

    def signup(self, name, password):
        if any(c.customer_name == name for c in self.customer_list):
            return None 
        new_customer = Customer(name, password)
        self.customer_list.append(new_customer)
        self.save_to_csv()
        return new_customer

    # ④ 購入データの保存
    def save_purchase(self, car):
        file_exists = os.path.exists(self.purchase_file)
        with open(self.purchase_file, mode="a", encoding="utf-8", newline="") as f:
            fieldnames = ["series", "color", "wheel", "interior", "engine"]
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            if not file_exists:
                writer.writeheader()
            writer.writerow({
                "series": car.series,
                "color": car.color,
                "wheel": car.wheel,
                "interior": car.interior,
                "engine": car.engine
            })

    # ⑤ 人気No.1のカスタムを各項目ごとに出す
    def get_popular_options(self, series_name: str) -> dict:
        result = {"color": None, "wheel": None, "interior": None, "engine": None}
        if not os.path.exists(self.purchase_file):
            return result
            
        colors, wheels, interiors, engines = [], [], [], []
        with open(self.purchase_file, mode="r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                if row["series"] == series_name:
                    colors.append(row["color"])
                    wheels.append(row["wheel"])
                    interiors.append(row["interior"])
                    engines.append(row["engine"])
        
        if colors: result["color"] = Counter(colors).most_common(1)[0][0]
        if wheels: result["wheel"] = Counter(wheels).most_common(1)[0][0]
        if interiors: result["interior"] = Counter(interiors).most_common(1)[0][0]
        if engines: result["engine"] = Counter(engines).most_common(1)[0][0]
        return result

```
### app.py
ログインおよび新規登録を行うためのアプリのエントリーポイントです。

```Python
import streamlit as st
from models.session_manager import SessionManager

st.set_page_config(page_title="Car Customizer", layout="centered")

if "manager" not in st.session_state:
    st.session_state.manager = SessionManager()
if "logged_in_user" not in st.session_state:
    st.session_state.logged_in_user = None

st.title("🚗 車種カスタマイズアプリ")

if st.session_state.logged_in_user:
    st.success(f"{st.session_state.logged_in_user.customer_name}さん、ログイン中です。")
    if st.button("マイページへ移動"):
        st.switch_page("pages/1_mypage.py")
else:
    tab1, tab2 = st.tabs(["ログイン", "新規登録"])
    
    with tab1:
        st.subheader("ログイン")
        l_name = st.text_input("お名前", key="l_name")
        l_pass = st.text_input("パスワード", type="password", key="l_pass")
        if st.button("ログイン"):
            user = st.session_state.manager.login(l_name, l_pass)
            if user:
                st.session_state.logged_in_user = user
                st.success("ログインしました！")
                st.switch_page("pages/1_mypage.py")
            else:
                st.error("名前またはパスワードが間違っています。")
                
    with tab2:
        st.subheader("新規アカウント作成")
        s_name = st.text_input("お名前", key="s_name")
        s_pass = st.text_input("パスワード", type="password", key="s_pass")
        if st.button("登録してログイン"):
            user = st.session_state.manager.signup(s_name, s_pass)
            if user:
                st.session_state.logged_in_user = user
                st.success("アカウントを作成しました！")
                st.switch_page("pages/1_mypage.py")
            else:
                st.error("そのお名前は既に登録されています。")
```

### pages/1_mypage.py
ユーザーのマイページ。保存されたカスタム履歴の表示と再開処理を行います。

```Python
import streamlit as st
from models.car import Car

if "logged_in_user" not in st.session_state or st.session_state.logged_in_user is None:
    st.warning("ログインしてください。")
    st.stop()

user = st.session_state.logged_in_user
st.title(f"🏠 {user.customer_name} さんのマイページ")

if st.sidebar.button("ログアウト"):
    st.session_state.manager.save_to_csv()
    st.session_state.logged_in_user = None
    st.switch_page("app.py")

st.subheader("新しい車をカスタマイズする")
if st.button("✨ 新規見積もりを始める"):
    user.customer_car = Car.create("カローラ")
    st.switch_page("pages/2_customize.py")

st.divider()

st.subheader("📜 履歴から続ける")
history_cars = user.get_history_instances()

if not history_cars:
    st.info("保存された履歴はありません。")
else:
    for i, car in enumerate(history_cars):
        # ① 履歴の表示にプレビュー画像（カラーボックス）も一緒に表示
        border_style = "border: 1px solid #ccc;" if car.get_color_code() in ["#ffffff", "#f8f9fa", "#f5f5f5"] else ""
        preview_html = f'''
        <div style="display: flex; align-items: center; gap: 15px; margin-bottom: 10px; padding: 10px; background-color: #f8f9fa; border-radius: 5px;">
            <div style="width: 50px; height: 30px; background-color: {car.get_color_code()}; border-radius: 4px; {border_style}"></div>
            <div><strong>{car.series}</strong> (¥{car.get_total_price():,})</div>
        </div>
        '''
        st.markdown(preview_html, unsafe_allow_html=True)
        
        with st.expander(f"履歴 {i+1} の構成詳細を確認"):
            st.write(f"- カラー: {car.color}")
            st.write(f"- ホイール: {car.wheel}")
            st.write(f"- 内装: {car.interior}")
            st.write(f"- エンジン: {car.engine}")
            
            if st.button("この構成から再開する", key=f"resume_{i}"):
                user.customer_car = car
                st.switch_page("pages/2_customize.py")
```

### pages/2_customize.py
実際のカスタマイズを行う画面です。ヘッダーの固定表示やPDFの動的生成ロジックを含みます。

```Python
import streamlit as st
import io
from models.car import Car
from reportlab.pdfgen import canvas
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.cidfonts import UnicodeCIDFont

if "logged_in_user" not in st.session_state or st.session_state.logged_in_user is None:
    st.warning("ログインしてください。")
    st.stop()

user = st.session_state.logged_in_user
car = user.customer_car

# ⑤ 人気No.1データの取得
popular_options = st.session_state.manager.get_popular_options(car.series)

# ② CSSによる画像と合計金額の画面上部への固定表示
st.markdown("""
    <style>
    /* 固定ヘッダーのスタイル */
    .sticky-header {
        position: fixed;
        top: 2.8rem;
        left: 0;
        width: 100%;
        background-color: rgba(255, 255, 255, 0.95);
        z-index: 99;
        padding: 10px 20px;
        border-bottom: 2px solid #f0f2f6;
        box-shadow: 0 2px 5px rgba(0,0,0,0.05);
    }
    /* メインコンテンツがヘッダーに隠れないようにするマージン */
    .main-body {
        margin-top: 140px;
    }
    </style>
""", unsafe_allow_html=True)

# 固定表示用コンテナ
border_style = "border: 1px solid #ccc;" if car.get_color_code() in ["#ffffff", "#f8f9fa", "#f5f5f5"] else ""
header_html = f'''
<div class="sticky-header">
    <div style="max-width: 700px; margin: 0 auto; display: flex; justify-content: space-between; align-items: center;">
        <div style="display: flex; align-items: center; gap: 20px;">
            <div style="width: 120px; height: 70px; background-color: {car.get_color_code()}; border-radius: 8px; {border_style} display: flex; align-items: center; justify-content: center; color: {'#000' if car.get_color_code() in ['#ffffff', '#f8f9fa', '#f5f5f5', '#e0a800'] else '#fff'}; font-weight: bold; font-size: 12px; box-shadow: inset 0 0 10px rgba(0,0,0,0.2);">
                {car.series}
            </div>
            <div>
                <h3 style="margin: 0; font-size: 18px; color: #333;">現在のカスタム状況</h3>
                <p style="margin: 5px 0 0 0; font-size: 13px; color: #666;">カラー: {car.color}</p>
            </div>
        </div>
        <div style="text-align: right;">
            <span style="font-size: 12px; color: #666; display: block;">現在の合計金額 (税込)</span>
            <span style="font-size: 24px; font-weight: bold; color: #ff4b4b;">¥ {car.get_total_price():,}</span>
        </div>
    </div>
</div>
<div class="main-body"></div>
'''
st.markdown(header_html, unsafe_allow_html=True)

st.title("🔧 カスタマイズ画面")

# サイドバーメニュー（⑤ 一時保存ボタン）
with st.sidebar:
    st.subheader("操作メニュー")
    if st.button("💾 現在の状態を一時保存", type="primary"):
        user.add_history(car.to_string())
        st.session_state.manager.save_to_csv()
        st.success("マイページの履歴に保存しました！")
    
    st.divider()
    if st.button("🔙 マイページへ戻る"):
        st.switch_page("pages/1_mypage.py")

# ③ PDF生成関数 (ReportLabの日本語標準フォント使用)
def generate_pdf_data(car, user):
    buffer = io.BytesIO()
    p = canvas.Canvas(buffer)
    
    # 日本語フォント登録
    pdfmetrics.registerFont(UnicodeCIDFont('HeiseiKakuGo-W5'))
    
    # 資料の描画
    p.setFont('HeiseiKakuGo-W5', 20)
    p.drawString(50, 750, "自動車カスタム 御見積書 (販売店控え)")
    
    p.setFont('HeiseiKakuGo-W5', 12)
    p.drawString(50, 710, f"お客様ID: {user.customer_id}")
    p.drawString(50, 690, f"お名前: {user.customer_name} 様")
    
    p.setLineWidth(1)
    p.line(50, 670, 550, 670)
    
    p.setFont('HeiseiKakuGo-W5', 14)
    p.drawString(50, 640, f"基本車種: {car.series}")
    p.drawRightString(550, 640, f"本体価格: ¥{car.series_price:,}")
    
    p.setFont('HeiseiKakuGo-W5', 12)
    y = 600
    options = [
        ("ボディカラー", car.color, car.color_price.get(car.color, 0)),
        ("ホイール", car.wheel, car.wheel_price.get(car.wheel, 0)),
        ("内装仕様", car.interior, car.interior_price.get(car.interior, 0)),
        ("エンジン", car.engine, car.engine_price.get(car.engine, 0)),
    ]
    
    for label, val, price in options:
        p.drawString(70, y, f"・{label}: {val}")
        p.drawRightString(550, y, f"+ ¥{price:,}")
        y -= 25
        
    p.line(50, y, 550, y)
    y -= 30
    
    p.setFont('HeiseiKakuGo-W5', 16)
    p.drawString(50, y, "総合計金額 (税込)")
    p.drawRightString(550, y, f"¥{car.get_total_price():,}")
    
    p.showPage()
    p.save()
    
    buffer.seek(0)
    return buffer.getvalue()

# ③ 購入ボタンと確認ダイアログの表示
@st.dialog("🛒 ご購入手続きの確認")
def show_purchase_dialog():
    st.write("以下のカスタム内容で注文確定および販売店伝達用PDF資料を発行します。")
    st.markdown(f"""
    <div style="background-color: #f8f9fa; padding: 15px; border-radius: 5px; border-left: 4px solid #ff4b4b; margin-bottom: 20px;">
        <strong>車種:</strong> {car.series}<br>
        <strong>カラー:</strong> {car.color}<br>
        <strong>ホイール:</strong> {car.wheel}<br>
        <strong>内装:</strong> {car.interior}<br>
        <strong>エンジン:</strong> {car.engine}<br>
        <hr style="margin: 10px 0;">
        <strong style="color: #ff4b4b; font-size: 18px;">合計金額: ¥{car.get_total_price():,}</strong>
    </div>
    """, unsafe_allow_html=True)
    
    # PDFデータの作成
    pdf_bytes = generate_pdf_data(car, user)
    
    # ④ 購入データの保存をコールバックで行う
    def confirm_purchase():
        st.session_state.manager.save_purchase(car)
        st.toast("⚡ 購入データが正常に保存され、PDFがダウンロードされました！")

    st.download_button(
        label="🔴 購入を確定してPDFをダウンロード",
        data=pdf_bytes,
        file_name=f"車注文申込書_{car.series}.pdf",
        mime="application/pdf",
        on_click=confirm_purchase,
        use_container_width=True
    )

# 画面下部に購入ボタンを配置
st.subheader("購入手続き")
if st.button("🛒 ご購入手続きへ進む", type="primary", use_container_width=True):
    show_purchase_dialog()

st.divider()

# ⑥ 車種の変更
st.subheader("1. 車種の変更")
series_list = Car.get_all_series_names()
def on_series_change():
    new_series = st.session_state.series_selector
    user.customer_car = Car.create(new_series)

current_index = series_list.index(car.series) if car.series in series_list else 0
st.selectbox("車種を選択してください", series_list, index=current_index, key="series_selector", on_change=on_series_change)

st.subheader("2. オプションのカスタマイズ")
col_a, col_b = st.columns(2)

# ⑤ 人気No.1を分かりやすくラベル表示するヘルパー関数
def get_label(option_name, popular_val):
    if option_name == popular_val:
        return f"{option_name} (🔥 人気No.1)"
    return option_name

with col_a:
    car.color = st.selectbox(
        "ボディカラー", 
        options=list(car.color_price.keys()),
        index=list(car.color_price.keys()).index(car.color) if car.color in car.color_price else 0,
        format_func=lambda x: f"{get_label(x, popular_options['color'])} (+¥{car.color_price[x]:,})"
    )
    
    car.wheel = st.selectbox(
        "ホイール", 
        options=list(car.wheel_price.keys()),
        index=list(car.wheel_price.keys()).index(car.wheel) if car.wheel in car.wheel_price else 0,
        format_func=lambda x: f"{get_label(x, popular_options['wheel'])} (+¥{car.wheel_price[x]:,})"
    )

with col_b:
    car.interior = st.selectbox(
        "インテリア", 
        options=list(car.interior_price.keys()),
        index=list(car.interior_price.keys()).index(car.interior) if car.interior in car.interior_price else 0,
        format_func=lambda x: f"{get_label(x, popular_options['interior'])} (+¥{car.interior_price[x]:,})"
    )
    
    car.engine = st.selectbox(
        "エンジン", 
        options=list(car.engine_price.keys()),
        index=list(car.engine_price.keys()).index(car.engine) if car.engine in car.engine_price else 0,
        format_func=lambda x: f"{get_label(x, popular_options['engine'])} (+¥{car.engine_price[x]:,})"
    )
```
