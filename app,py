import streamlit as st
import sqlite3
import hashlib
from datetime import datetime, date, time, timedelta
from contextlib import contextmanager

# =========================================================
# CONFIG
# =========================================================

st.set_page_config(
    page_title="Tour Operations Hub",
    page_icon="🚌",
    layout="wide",
    initial_sidebar_state="expanded",
)

DB_FILE = "tour_operations.db"


# =========================================================
# DATABASE
# =========================================================

@contextmanager
def get_db():
    conn = sqlite3.connect(DB_FILE, check_same_thread=False)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    finally:
        conn.close()


def hash_password(password):
    return hashlib.sha256(password.encode()).hexdigest()


def init_db():
    with get_db() as conn:
        cur = conn.cursor()

        cur.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                password TEXT NOT NULL,
                full_name TEXT NOT NULL,
                role TEXT NOT NULL,
                phone TEXT,
                email TEXT,
                active INTEGER DEFAULT 1,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS guides (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT UNIQUE NOT NULL,
                full_name TEXT NOT NULL,
                phone TEXT,
                email TEXT,
                languages TEXT,
                experience INTEGER DEFAULT 0,
                specialties TEXT,
                status TEXT DEFAULT 'Đang hoạt động',
                notes TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS tours (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT UNIQUE NOT NULL,
                name TEXT NOT NULL,
                destination TEXT,
                departure_date TEXT NOT NULL,
                return_date TEXT NOT NULL,
                departure_time TEXT,
                meeting_point TEXT,
                customer_count INTEGER DEFAULT 0,
                tour_type TEXT,
                status TEXT DEFAULT 'Sắp khởi hành',
                operator_name TEXT,
                operator_phone TEXT,
                description TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS assignments (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                tour_id INTEGER NOT NULL,
                guide_id INTEGER NOT NULL,
                role TEXT DEFAULT 'HDV chính',
                status TEXT DEFAULT 'Chờ xác nhận',
                assigned_at TEXT DEFAULT CURRENT_TIMESTAMP,
                confirmed_at TEXT,
                note TEXT,
                UNIQUE(tour_id, guide_id),
                FOREIGN KEY(tour_id) REFERENCES tours(id),
                FOREIGN KEY(guide_id) REFERENCES guides(id)
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS itineraries (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                tour_id INTEGER NOT NULL,
                day_number INTEGER NOT NULL,
                itinerary_date TEXT,
                start_time TEXT,
                end_time TEXT,
                title TEXT NOT NULL,
                location TEXT,
                description TEXT,
                FOREIGN KEY(tour_id) REFERENCES tours(id)
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS notifications (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                title TEXT NOT NULL,
                message TEXT NOT NULL,
                notification_type TEXT DEFAULT 'info',
                is_read INTEGER DEFAULT 0,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(user_id) REFERENCES users(id)
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS incidents (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                tour_id INTEGER NOT NULL,
                guide_id INTEGER,
                incident_type TEXT,
                description TEXT,
                severity TEXT DEFAULT 'Trung bình',
                status TEXT DEFAULT 'Mới',
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(tour_id) REFERENCES tours(id),
                FOREIGN KEY(guide_id) REFERENCES guides(id)
            )
        """)

        cur.execute("""
            CREATE TABLE IF NOT EXISTS tour_reports (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                tour_id INTEGER NOT NULL,
                guide_id INTEGER,
                summary TEXT,
                customer_feedback TEXT,
                guide_feedback TEXT,
                completed_at TEXT DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(tour_id) REFERENCES tours(id),
                FOREIGN KEY(guide_id) REFERENCES guides(id)
            )
        """)

        # Tạo tài khoản mẫu
        admin = cur.execute(
            "SELECT id FROM users WHERE username=?",
            ("admin",)
        ).fetchone()

        if not admin:
            cur.execute("""
                INSERT INTO users
                (username, password, full_name, role, phone, email)
                VALUES (?, ?, ?, ?, ?, ?)
            """, (
                "admin",
                hash_password("admin123"),
                "Quản trị viên",
                "ADMIN",
                "0900000000",
                "admin@company.com"
            ))

        guide_user = cur.execute(
            "SELECT id FROM users WHERE username=?",
            ("hdv01",)
        ).fetchone()

        if not guide_user:
            cur.execute("""
                INSERT INTO users
                (username, password, full_name, role, phone, email)
                VALUES (?, ?, ?, ?, ?, ?)
            """, (
                "hdv01",
                hash_password("123456"),
                "Nguyễn Văn An",
                "HDV",
                "0912345678",
                "an@example.com"
            ))

        # HDV mẫu
        guide = cur.execute(
            "SELECT id FROM guides WHERE code=?",
            ("HDV001",)
        ).fetchone()

        if not guide:
            cur.execute("""
                INSERT INTO guides
                (code, full_name, phone, email, languages,
                 experience, specialties, status)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                "HDV001",
                "Nguyễn Văn An",
                "0912345678",
                "an@example.com",
                "Tiếng Việt, English",
                5,
                "Tour nội địa, Đà Lạt, Vũng Tàu",
                "Đang hoạt động"
            ))


# =========================================================
# HELPERS
# =========================================================

def query(sql, params=(), fetch=False, many=False):
    with get_db() as conn:
        cur = conn.cursor()

        if many:
            cur.executemany(sql, params)
        else:
            cur.execute(sql, params)

        if fetch:
            return cur.fetchall()

        return cur.lastrowid


def execute(sql, params=()):
    with get_db() as conn:
        conn.execute(sql, params)


def format_date(value):
    if not value:
        return ""
    try:
        return datetime.strptime(value, "%Y-%m-%d").strftime("%d/%m/%Y")
    except:
        return value


def days_between(start, end):
    try:
        d1 = datetime.strptime(start, "%Y-%m-%d")
        d2 = datetime.strptime(end, "%Y-%m-%d")
        return (d2 - d1).days + 1
    except:
        return 0


def status_badge(status):
    mapping = {
        "Đã xác nhận": "🟢",
        "Chờ xác nhận": "🟡",
        "Sắp khởi hành": "🔵",
        "Đang thực hiện": "🟣",
        "Đã hoàn thành": "⚫",
        "Đã hủy": "🔴",
        "Đang hoạt động": "🟢",
        "Tạm nghỉ": "🟡",
        "Mới": "🔴",
        "Đang xử lý": "🟡",
        "Đã xử lý": "🟢",
    }

    return f"{mapping.get(status, '⚪')} {status}"


def is_admin():
    return st.session_state.get("role") in ["ADMIN", "OPERATOR", "MANAGER"]


def current_user():
    return st.session_state.get("user")


# =========================================================
# AUTHENTICATION
# =========================================================

def login_page():

    st.markdown("""
        <style>
        .login-box {
            max-width: 500px;
            margin: 80px auto;
            padding: 30px;
            border-radius: 18px;
            background: #ffffff;
            box-shadow: 0 5px 30px rgba(0,0,0,.08);
        }
        </style>
    """, unsafe_allow_html=True)

    st.markdown(
        '<div class="login-box">',
        unsafe_allow_html=True
    )

    st.markdown("# 🚌 Tour Operations Hub")
    st.caption("Hệ thống điều hành tour nội bộ")

    username = st.text_input(
        "👤 Tài khoản",
        placeholder="Nhập tài khoản"
    )

    password = st.text_input(
        "🔒 Mật khẩu",
        type="password",
        placeholder="Nhập mật khẩu"
    )

    if st.button(
        "ĐĂNG NHẬP",
        use_container_width=True,
        type="primary"
    ):
        user = query("""
            SELECT *
            FROM users
            WHERE username=?
            AND password=?
            AND active=1
        """, (
            username,
            hash_password(password)
        ), True)

        if user:
            u = user[0]

            st.session_state.logged_in = True
            st.session_state.user = dict(u)
            st.session_state.role = u["role"]

            st.success("Đăng nhập thành công!")
            st.rerun()

        else:
            st.error("Sai tài khoản hoặc mật khẩu.")

    st.info("""
    **Tài khoản demo**

    Admin:
    `admin` / `admin123`

    HDV:
    `hdv01` / `123456`
    """)

    st.markdown("</div>", unsafe_allow_html=True)


# =========================================================
# SIDEBAR
# =========================================================

def sidebar():

    user = current_user()

    with st.sidebar:

        st.markdown("# 🚌 TOUR HUB")
        st.caption("Hệ thống điều hành nội bộ")

        st.divider()

        st.markdown(f"""
        **{user['full_name']}**

        Vai trò: `{user['role']}`
        """)

        st.divider()

        if is_admin():

            menu = st.radio(
                "MENU",
                [
                    "🏠 Dashboard",
                    "📅 Lịch phân công",
                    "🚌 Quản lý tour",
                    "👨‍💼 Quản lý HDV",
                    "👥 Khách đoàn",
                    "🔔 Thông báo",
                    "🚨 Sự cố",
                    "📊 Báo cáo",
                ]
            )

        else:

            menu = st.radio(
                "MENU",
                [
                    "🏠 Dashboard",
                    "📅 Lịch tour của tôi",
                    "🚌 Tour của tôi",
                    "🗺️ Lịch trình",
                    "🔔 Thông báo",
                    "🚨 Báo sự cố",
                    "📝 Báo cáo tour",
                ]
            )

        st.divider()

        if st.button(
            "🚪 Đăng xuất",
            use_container_width=True
        ):
            for key in list(st.session_state.keys()):
                del st.session_state[key]

            st.rerun()

    return menu


# =========================================================
# DASHBOARD
# =========================================================

def dashboard():

    st.title("🏠 Dashboard")

    today = date.today().isoformat()

    tours = query(
        "SELECT * FROM tours",
        fetch=True
    )

    guides = query(
        "SELECT * FROM guides",
        fetch=True
    )

    if is_admin():

        total_tours = len(tours)
        active_guides = len([
            g for g in guides
            if g["status"] == "Đang hoạt động"
        ])

        upcoming = len([
            t for t in tours
            if t["departure_date"] >= today
            and t["status"] != "Đã hủy"
        ])

        completed = len([
            t for t in tours
            if t["status"] == "Đã hoàn thành"
        ])

        c1, c2, c3, c4 = st.columns(4)

        c1.metric("🚌 Tổng tour", total_tours)
        c2.metric("👨‍💼 HDV hoạt động", active_guides)
        c3.metric("📅 Tour sắp tới", upcoming)
        c4.metric("✅ Tour hoàn thành", completed)

    else:

        guide = query(
            "SELECT * FROM guides WHERE full_name=?",
            (current_user()["full_name"],),
            True
        )

        if guide:
            guide_id = guide[0]["id"]

            assignments = query("""
                SELECT a.*, t.*
                FROM assignments a
                JOIN tours t ON t.id=a.tour_id
                WHERE a.guide_id=?
                ORDER BY t.departure_date
            """, (guide_id,), True)

        else:
            assignments = []

        upcoming = [
            x for x in assignments
            if x["departure_date"] >= today
            and x["status"] != "Đã hủy"
        ]

        completed = [
            x for x in assignments
            if x["status"] == "Đã hoàn thành"
        ]

        pending = [
            x for x in assignments
            if x["status"] == "Chờ xác nhận"
        ]

        c1, c2, c3, c4 = st.columns(4)

        c1.metric("🚌 Tour của tôi", len(assignments))
        c2.metric("📅 Tour sắp tới", len(upcoming))
        c3.metric("🟡 Chờ xác nhận", len(pending))
        c4.metric("✅ Đã hoàn thành", len(completed))

    st.divider()

    st.subheader("📅 Tour sắp khởi hành")

    upcoming_tours = [
        t for t in tours
        if t["departure_date"] >= today
        and t["status"] != "Đã hủy"
    ]

    upcoming_tours = sorted(
        upcoming_tours,
        key=lambda x: x["departure_date"]
    )[:10]

    if not upcoming_tours:
        st.info("Chưa có tour sắp khởi hành.")
        return

    for t in upcoming_tours:

        with st.container(border=True):

            c1, c2, c3 = st.columns([2, 3, 2])

            c1.markdown(
                f"### 🚌 {t['code']}"
            )

            c1.write(t["name"])

            c2.write(
                f"📅 {format_date(t['departure_date'])} → "
                f"{format_date(t['return_date'])}"
            )

            c2.write(
                f"📍 {t['destination']}  •  "
                f"👥 {t['customer_count']} khách"
            )

            c3.write(status_badge(t["status"]))


# =========================================================
# TOUR MANAGEMENT
# =========================================================

def tour_management():

    st.title("🚌 Quản lý tour")

    tabs = st.tabs([
        "📋 Danh sách tour",
        "➕ Tạo tour",
        "🗺️ Lịch trình",
    ])

    with tabs[0]:

        tours = query("""
            SELECT *
            FROM tours
            ORDER BY departure_date DESC
        """, fetch=True)

        search = st.text_input(
            "🔎 Tìm tour",
            placeholder="Mã tour, tên tour, điểm đến..."
        )

        filtered = tours

        if search:
            search_lower = search.lower()

            filtered = [
                t for t in tours
                if search_lower in str(t["code"]).lower()
                or search_lower in str(t["name"]).lower()
                or search_lower in str(t["destination"]).lower()
            ]

        for t in filtered:

            with st.container(border=True):

                c1, c2, c3, c4 = st.columns(
                    [1.2, 3, 2.5, 1.5]
                )

                c1.markdown(
                    f"**{t['code']}**"
                )

                c2.markdown(
                    f"**{t['name']}**"
                )

                c2.caption(
                    f"📍 {t['destination']}"
                )

                c3.write(
                    f"📅 {format_date(t['departure_date'])}"
                    f" → "
                    f"{format_date(t['return_date'])}"
                )

                c3.write(
                    f"👥 {t['customer_count']} khách"
                )

                c4.write(
                    status_badge(t["status"])
                )

                if st.button(
                    "Xem chi tiết",
                    key=f"tour_{t['id']}"
                ):
                    st.session_state.selected_tour = t["id"]

        if "selected_tour" in st.session_state:

            tour_detail(
                st.session_state.selected_tour
            )

    with tabs[1]:

        create_tour()

    with tabs[2]:

        itinerary_management()


def create_tour():

    st.subheader("➕ Tạo tour mới")

    with st.form("create_tour"):

        c1, c2 = st.columns(2)

        code = c1.text_input(
            "Mã tour *",
            placeholder="DL3N2D-001"
        )

        name = c2.text_input(
            "Tên tour *",
            placeholder="Đà Lạt 3 ngày 2 đêm"
        )

        destination = c1.text_input(
            "Điểm đến"
        )

        tour_type = c2.selectbox(
            "Loại tour",
            [
                "Nội địa",
                "Quốc tế",
                "Nghỉ dưỡng",
                "Gia đình",
                "MICE",
                "Khác"
            ]
        )

        departure_date = c1.date_input(
            "Ngày khởi hành",
            date.today()
        )

        return_date = c2.date_input(
            "Ngày kết thúc",
            date.today() + timedelta(days=2)
        )

        departure_time = c1.time_input(
            "Giờ khởi hành",
            time(5, 30)
        )

        customer_count = c2.number_input(
            "Số lượng khách",
            min_value=0,
            value=20
        )

        meeting_point = st.text_input(
            "📍 Điểm tập trung"
        )

        operator_name = st.text_input(
            "👨‍💼 Điều hành phụ trách"
        )

        operator_phone = st.text_input(
            "📞 SĐT điều hành"
        )

        description = st.text_area(
            "Ghi chú"
        )

        submitted = st.form_submit_button(
            "💾 TẠO TOUR",
            type="primary",
            use_container_width=True
        )

        if submitted:

            if not code or not name:
                st.error(
                    "Vui lòng nhập mã tour và tên tour."
                )

            elif return_date < departure_date:
                st.error(
                    "Ngày kết thúc không thể trước ngày khởi hành."
                )

            else:

                try:

                    execute("""
                        INSERT INTO tours
                        (
                            code, name, destination,
                            departure_date, return_date,
                            departure_time,
                            meeting_point,
                            customer_count,
                            tour_type,
                            operator_name,
                            operator_phone,
                            description
                        )
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                    """, (
                        code,
                        name,
                        destination,
                        departure_date.isoformat(),
                        return_date.isoformat(),
                        departure_time.strftime("%H:%M"),
                        meeting_point,
                        customer_count,
                        tour_type,
                        operator_name,
                        operator_phone,
                        description
                    ))

                    st.success(
                        "Tạo tour thành công!"
                    )

                except sqlite3.IntegrityError:
                    st.error(
                        "Mã tour đã tồn tại."
                    )


# =========================================================
# TOUR DETAIL
# =========================================================

def tour_detail(tour_id):

    tour = query(
        "SELECT * FROM tours WHERE id=?",
        (tour_id,),
        True
    )

    if not tour:
        return

    tour = tour[0]

    st.divider()

    st.subheader(
        f"🚌 {tour['code']} – {tour['name']}"
    )

    c1, c2, c3, c4 = st.columns(4)

    c1.metric(
        "📅 Khởi hành",
        format_date(tour["departure_date"])
    )

    c2.metric(
        "📅 Kết thúc",
        format_date(tour["return_date"])
    )

    c3.metric(
        "👥 Khách",
        tour["customer_count"]
    )

    c4.metric(
        "⏱ Số ngày",
        days_between(
            tour["departure_date"],
            tour["return_date"]
        )
    )

    st.write(
        f"**📍 Điểm đến:** {tour['destination']}"
    )

    st.write(
        f"**📍 Điểm tập trung:** {tour['meeting_point']}"
    )

    st.write(
        f"**⏰ Giờ khởi hành:** {tour['departure_time']}"
    )

    st.write(
        f"**👨‍💼 Điều hành:** {tour['operator_name']} "
        f" – {tour['operator_phone']}"
    )

    st.write(
        f"**Trạng thái:** {status_badge(tour['status'])}"
    )

    if tour["description"]:
        st.info(tour["description"])

    if is_admin():

        new_status = st.selectbox(
            "Cập nhật trạng thái",
            [
                "Sắp khởi hành",
                "Đang thực hiện",
                "Đã hoàn thành",
                "Đã hủy"
            ],
            index=[
                "Sắp khởi hành",
                "Đang thực hiện",
                "Đã hoàn thành",
                "Đã hủy"
            ].index(tour["status"])
            if tour["status"] in [
                "Sắp khởi hành",
                "Đang thực hiện",
                "Đã hoàn thành",
                "Đã hủy"
            ]
            else 0
        )

        if st.button(
            "💾 Lưu trạng thái",
            key=f"status_{tour_id}"
        ):

            execute(
                "UPDATE tours SET status=? WHERE id=?",
                (new_status, tour_id)
            )

            st.success(
                "Đã cập nhật trạng thái."
            )

            st.rerun()

    st.markdown("### 👨‍💼 HDV được phân công")

    assignments = query("""
        SELECT
            a.*,
            g.code,
            g.full_name,
            g.phone,
            g.languages
        FROM assignments a
        JOIN guides g ON g.id=a.guide_id
        WHERE a.tour_id=?
    """, (tour_id,), True)

    if assignments:

        for a in assignments:

            with st.container(border=True):

                c1, c2, c3, c4 = st.columns(4)

                c1.write(
                    f"**{a['code']}**"
                )

                c2.write(
                    f"👤 {a['full_name']}"
                )

                c3.write(
                    f"Vai trò: {a['role']}"
                )

                c4.write(
                    status_badge(a["status"])
                )

                if a["status"] == "Chờ xác nhận":

                    if st.button(
                        "✅ Xác nhận",
                        key=f"confirm_{a['id']}"
                    ):

                        execute("""
                            UPDATE assignments
                            SET status='Đã xác nhận',
                                confirmed_at=?
                            WHERE id=?
                        """, (
                            datetime.now().isoformat(),
                            a["id"]
                        ))

                        st.success(
                            "Đã xác nhận tour."
                        )

                        st.rerun()

    else:

        st.info(
            "Tour chưa được phân công HDV."
        )


# =========================================================
# GUIDE MANAGEMENT
# =========================================================

def guide_management():

    st.title("👨‍💼 Quản lý hướng dẫn viên")

    tabs = st.tabs([
        "👥 Danh sách HDV",
        "➕ Thêm HDV",
        "📅 Phân công tour"
    ])

    with tabs[0]:

        guides = query("""
            SELECT *
            FROM guides
            ORDER BY full_name
        """, fetch=True)

        search = st.text_input(
            "🔎 Tìm HDV"
        )

        if search:

            search = search.lower()

            guides = [
                g for g in guides
                if search in g["full_name"].lower()
                or search in g["code"].lower()
            ]

        for g in guides:

            with st.container(border=True):

                c1, c2, c3, c4 = st.columns(
                    [1, 2.5, 2.5, 1.5]
                )

                c1.write(
                    f"**{g['code']}**"
                )

                c2.write(
                    f"👤 **{g['full_name']}**"
                )

                c2.caption(
                    f"📞 {g['phone']}"
                )

                c3.write(
                    f"🌐 {g['languages']}"
                )

                c3.write(
                    f"⭐ {g['experience']} năm"
                )

                c4.write(
                    status_badge(g["status"])
                )

    with tabs[1]:

        add_guide()

    with tabs[2]:

        assignment_page()


def add_guide():

    st.subheader("➕ Thêm hướng dẫn viên")

    with st.form("add_guide"):

        c1, c2 = st.columns(2)

        code = c1.text_input(
            "Mã HDV *",
            placeholder="HDV002"
        )

        name = c2.text_input(
            "Họ tên *"
        )

        phone = c1.text_input(
            "Số điện thoại"
        )

        email = c2.text_input(
            "Email"
        )

        languages = c1.text_input(
            "Ngôn ngữ",
            placeholder="Tiếng Việt, English"
        )

        experience = c2.number_input(
            "Kinh nghiệm (năm)",
            min_value=0,
            value=0
        )

        specialties = st.text_input(
            "Chuyên tour",
            placeholder="Đà Lạt, Phú Quốc, MICE..."
        )

        status = st.selectbox(
            "Trạng thái",
            [
                "Đang hoạt động",
                "Tạm nghỉ"
            ]
        )

        notes = st.text_area(
            "Ghi chú"
        )

        if st.form_submit_button(
            "💾 THÊM HDV",
            type="primary",
            use_container_width=True
        ):

            if not code or not name:
                st.error(
                    "Vui lòng nhập mã và tên HDV."
                )
                return

            try:

                execute("""
                    INSERT INTO guides
                    (
                        code, full_name, phone,
                        email, languages,
                        experience, specialties,
                        status, notes
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, (
                    code,
                    name,
                    phone,
                    email,
                    languages,
                    experience,
                    specialties,
                    status,
                    notes
                ))

                st.success(
                    "Thêm HDV thành công!"
                )

            except sqlite3.IntegrityError:

                st.error(
                    "Mã HDV đã tồn tại."
                )


# =========================================================
# ASSIGNMENT
# =========================================================

def assignment_page():

    st.subheader("📅 Phân công HDV cho tour")

    tours = query("""
        SELECT *
        FROM tours
        WHERE status != 'Đã hủy'
        ORDER BY departure_date
    """, fetch=True)

    guides = query("""
        SELECT *
        FROM guides
        WHERE status='Đang hoạt động'
        ORDER BY full_name
    """, fetch=True)

    if not tours or not guides:
        st.warning(
            "Cần có tour và HDV trước khi phân công."
        )
        return

    tour_options = {
        f"{t['code']} – {t['name']} "
        f"({format_date(t['departure_date'])})": t["id"]
        for t in tours
    }

    guide_options = {
        f"{g['code']} – {g['full_name']}": g["id"]
        for g in guides
    }

    selected_tour_label = st.selectbox(
        "🚌 Chọn tour",
        list(tour_options.keys())
    )

    selected_tour_id = tour_options[
        selected_tour_label
    ]

    selected_tour = [
        t for t in tours
        if t["id"] == selected_tour_id
    ][0]

    st.info(
        f"📅 {format_date(selected_tour['departure_date'])}"
        f" → "
        f"{format_date(selected_tour['return_date'])}"
    )

    selected_guide_label = st.selectbox(
        "👨‍💼 Chọn HDV",
        list(guide_options.keys())
    )

    selected_guide_id = guide_options[
        selected_guide_label
    ]

    role = st.selectbox(
        "Vai trò",
        [
            "HDV chính",
            "HDV phụ"
        ]
    )

    note = st.text_area(
        "Ghi chú"
    )

    # Kiểm tra trùng lịch
    conflicts = query("""
        SELECT
            t.code,
            t.name,
            t.departure_date,
            t.return_date
        FROM assignments a
        JOIN tours t ON t.id=a.tour_id
        WHERE a.guide_id=?
        AND t.id != ?
        AND t.status != 'Đã hủy'
        AND t.departure_date <= ?
        AND t.return_date >= ?
    """, (
        selected_guide_id,
        selected_tour_id,
        selected_tour["return_date"],
        selected_tour["departure_date"]
    ), True)

    if conflicts:

        st.error(
            "⚠️ HDV này đang bị TRÙNG LỊCH với:"
        )

        for c in conflicts:

            st.write(
                f"- **{c['code']} – {c['name']}**: "
                f"{format_date(c['departure_date'])}"
                f" → "
                f"{format_date(c['return_date'])}"
            )

    else:

        st.success(
            "✅ Không phát hiện trùng lịch."
        )

    if st.button(
        "📌 PHÂN CÔNG HDV",
        type="primary",
        use_container_width=True
    ):

        if conflicts:

            st.error(
                "Không thể phân công do HDV bị trùng lịch."
            )

        else:

            try:

                execute("""
                    INSERT INTO assignments
                    (
                        tour_id,
                        guide_id,
                        role,
                        status,
                        note
                    )
                    VALUES (?, ?, ?, ?, ?)
                """, (
                    selected_tour_id,
                    selected_guide_id,
                    role,
                    "Chờ xác nhận",
                    note
                ))

                st.success(
                    "Phân công HDV thành công!"
                )

            except sqlite3.IntegrityError:

                st.warning(
                    "HDV này đã được phân công cho tour."
                )


# =========================================================
# MY TOUR
# =========================================================

def my_tours():

    st.title("🚌 Tour của tôi")

    guide = query(
        "SELECT * FROM guides WHERE full_name=?",
        (current_user()["full_name"],),
        True
    )

    if not guide:

        st.warning(
            "Chưa tìm thấy hồ sơ HDV."
        )
        return

    guide_id = guide[0]["id"]

    assignments = query("""
        SELECT
            a.*,
            t.*
        FROM assignments a
        JOIN tours t ON t.id=a.tour_id
        WHERE a.guide_id=?
        ORDER BY t.departure_date
    """, (guide_id,), True)

    if not assignments:

        st.info(
            "Bạn chưa được phân công tour nào."
        )
        return

    for a in assignments:

        with st.container(border=True):

            c1, c2, c3 = st.columns([1.5, 4, 2])

            c1.markdown(
                f"### {a['code']}"
            )

            c2.markdown(
                f"**{a['name']}**"
            )

            c2.write(
                f"📍 {a['destination']}"
            )

            c2.write(
                f"📅 {format_date(a['departure_date'])}"
                f" → "
                f"{format_date(a['return_date'])}"
            )

            c3.write(
                status_badge(a["status"])
            )

            if a["status"] == "Chờ xác nhận":

                if st.button(
                    "✅ Nhận tour",
                    key=f"accept_{a['id']}"
                ):

                    execute("""
                        UPDATE assignments
                        SET status='Đã xác nhận',
                            confirmed_at=?
                        WHERE id=?
                    """, (
                        datetime.now().isoformat(),
                        a["id"]
                    ))

                    st.success(
                        "Đã xác nhận nhận tour."
                    )

                    st.rerun()


# =========================================================
# CALENDAR
# =========================================================

def calendar_page():

    st.title("📅 Lịch phân công")

    if is_admin():

        assignments = query("""
            SELECT
                a.*,
                t.code,
                t.name,
                t.departure_date,
                t.return_date,
                g.full_name
            FROM assignments a
            JOIN tours t ON t.id=a.tour_id
            JOIN guides g ON g.id=a.guide_id
            ORDER BY t.departure_date
        """, fetch=True)

    else:

        guide = query(
            "SELECT * FROM guides WHERE full_name=?",
            (current_user()["full_name"],),
            True
        )

        if not guide:
            assignments = []
        else:
            assignments = query("""
                SELECT
                    a.*,
                    t.code,
                    t.name,
                    t.departure_date,
                    t.return_date,
                    g.full_name
                FROM assignments a
                JOIN tours t ON t.id=a.tour_id
                JOIN guides g ON g.id=a.guide_id
                WHERE a.guide_id=?
                ORDER BY t.departure_date
            """, (guide[0]["id"],), True)

    if not assignments:

        st.info(
            "Chưa có dữ liệu phân công."
        )
        return

    st.subheader("📋 Lịch tour")

    for a in assignments:

        with st.container(border=True):

            c1, c2, c3, c4 = st.columns(
                [1.2, 3, 2.5, 1.5]
            )

            c1.write(
                f"**{a['code']}**"
            )

            c2.write(
                f"🚌 {a['name']}"
            )

            c2.caption(
                f"👤 HDV: {a['full_name']}"
            )

            c3.write(
                f"📅 {format_date(a['departure_date'])}"
            )

            c3.write(
                f"→ {format_date(a['return_date'])}"
            )

            c4.write(
                status_badge(a["status"])
            )


# =========================================================
# ITINERARY
# =========================================================

def itinerary_management():

    st.subheader("🗺️ Lịch trình tour")

    tours = query("""
        SELECT *
        FROM tours
        ORDER BY departure_date DESC
    """, fetch=True)

    if not tours:
        st.info("Chưa có tour.")
        return

    options = {
        f"{t['code']} – {t['name']}": t["id"]
        for t in tours
    }

    selected = st.selectbox(
        "Chọn tour",
        list(options.keys())
    )

    tour_id = options[selected]

    items = query("""
        SELECT *
        FROM itineraries
        WHERE tour_id=?
        ORDER BY day_number, start_time
    """, (tour_id,), True)

    if items:

        current_day = None

        for item in items:

            if current_day != item["day_number"]:

                current_day = item["day_number"]

                st.markdown(
                    f"### 📅 Ngày {current_day}"
                )

            with st.container(border=True):

                c1, c2, c3 = st.columns(
                    [1, 3, 2]
                )

                c1.write(
                    f"⏰ {item['start_time']}"
                )

                c2.markdown(
                    f"**{item['title']}**"
                )

                c2.write(
                    item["description"]
                )

                c3.write(
                    f"📍 {item['location']}"
                )

    else:

        st.info(
            "Tour chưa có lịch trình."
        )

    st.divider()

    st.markdown("### ➕ Thêm hoạt động")

    with st.form(
        f"add_itinerary_{tour_id}"
    ):

        c1, c2, c3 = st.columns(3)

        day_number = c1.number_input(
            "Ngày thứ",
            min_value=1,
            value=1
        )

        start_time = c2.text_input(
            "Giờ bắt đầu",
            value="08:00"
        )

        end_time = c3.text_input(
            "Giờ kết thúc",
            value="09:00"
        )

        title = st.text_input(
            "Tên hoạt động *"
        )

        location = st.text_input(
            "Địa điểm"
        )

        description = st.text_area(
            "Mô tả"
        )

        if st.form_submit_button(
            "💾 Thêm lịch trình",
            type="primary"
        ):

            if not title:

                st.error(
                    "Vui lòng nhập tên hoạt động."
                )

            else:

                tour = query(
                    "SELECT departure_date FROM tours WHERE id=?",
                    (tour_id,),
                    True
                )

                start = datetime.strptime(
                    tour[0]["departure_date"],
                    "%Y-%m-%d"
                ).date()

                itinerary_date = (
                    start +
                    timedelta(days=int(day_number) - 1)
                ).isoformat()

                execute("""
                    INSERT INTO itineraries
                    (
                        tour_id,
                        day_number,
                        itinerary_date,
                        start_time,
                        end_time,
                        title,
                        location,
                        description
                    )
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                """, (
                    tour_id,
                    day_number,
                    itinerary_date,
                    start_time,
                    end_time,
                    title,
                    location,
                    description
                ))

                st.success(
                    "Đã thêm lịch trình."
                )

                st.rerun()


# =========================================================
# NOTIFICATIONS
# =========================================================

def notifications_page():

    st.title("🔔 Thông báo")

    user_id = current_user()["id"]

    notifications = query("""
        SELECT *
        FROM notifications
        WHERE user_id=?
        ORDER BY created_at DESC
    """, (user_id,), True)

    if not notifications:

        st.info(
            "Không có thông báo."
        )
        return

    for n in notifications:

        icon = {
            "info": "🔵",
            "warning": "🟡",
            "success": "🟢",
            "danger": "🔴"
        }.get(
            n["notification_type"],
            "🔵"
        )

        with st.container(border=True):

            st.markdown(
                f"### {icon} {n['title']}"
            )

            st.write(
                n["message"]
            )

            st.caption(
                n["created_at"]
            )

        if not n["is_read"]:

            execute(
                "UPDATE notifications SET is_read=1 WHERE id=?",
                (n["id"],)
            )


# =========================================================
# INCIDENTS
# =========================================================

def incidents_page():

    st.title("🚨 Báo sự cố")

    tours = query("""
        SELECT
            t.*
        FROM tours t
        JOIN assignments a
        ON a.tour_id=t.id
        JOIN guides g
        ON g.id=a.guide_id
        WHERE g.full_name=?
        ORDER BY t.departure_date DESC
    """, (current_user()["full_name"],), True)

    if not tours:

        st.info(
            "Bạn chưa có tour để báo sự cố."
        )
        return

    options = {
        f"{t['code']} – {t['name']}": t["id"]
        for t in tours
    }

    with st.form("incident_form"):

        tour_label = st.selectbox(
            "🚌 Tour",
            list(options.keys())
        )

        tour_id = options[tour_label]

        incident_type = st.selectbox(
            "Loại sự cố",
            [
                "Khách bị ốm",
                "Khách thất lạc",
                "Xe hỏng",
                "Trễ lịch trình",
                "Khách khiếu nại",
                "Khách sạn",
                "Nhà hàng",
                "Thời tiết",
                "Khác"
            ]
        )

        severity = st.selectbox(
            "Mức độ",
            [
                "Thấp",
                "Trung bình",
                "Cao",
                "Khẩn cấp"
            ]
        )

        description = st.text_area(
            "Mô tả chi tiết *"
        )

        if st.form_submit_button(
            "🚨 GỬI BÁO CÁO",
            type="primary"
        ):

            if not description:

                st.error(
                    "Vui lòng mô tả sự cố."
                )

            else:

                guide = query(
                    "SELECT id FROM guides WHERE full_name=?",
                    (current_user()["full_name"],),
                    True
                )

                guide_id = (
                    guide[0]["id"]
                    if guide
                    else None
                )

                execute("""
                    INSERT INTO incidents
                    (
                        tour_id,
                        guide_id,
                        incident_type,
                        description,
                        severity
                    )
                    VALUES (?, ?, ?, ?, ?)
                """, (
                    tour_id,
                    guide_id,
                    incident_type,
                    description,
                    severity
                ))

                st.success(
                    "Đã gửi báo cáo sự cố đến bộ phận điều hành."
                )


# =========================================================
# REPORT
# =========================================================

def report_page():

    st.title("📝 Báo cáo sau tour")

    guide = query(
        "SELECT id FROM guides WHERE full_name=?",
        (current_user()["full_name"],),
        True
    )

    if not guide:
        st.warning("Không tìm thấy hồ sơ HDV.")
        return

    tours = query("""
        SELECT DISTINCT t.*
        FROM tours t
        JOIN assignments a ON a.tour_id=t.id
        WHERE a.guide_id=?
        AND a.status='Đã xác nhận'
        ORDER BY t.departure_date DESC
    """, (guide[0]["id"],), True)

    if not tours:

        st.info(
            "Chưa có tour để báo cáo."
        )
        return

    options = {
        f"{t['code']} – {t['name']}": t["id"]
        for t in tours
    }

    with st.form("tour_report"):

        selected = st.selectbox(
            "Chọn tour",
            list(options.keys())
        )

        tour_id = options[selected]

        summary = st.text_area(
            "📝 Tóm tắt quá trình thực hiện tour"
        )

        customer_feedback = st.text_area(
            "👥 Phản hồi của khách"
        )

        guide_feedback = st.text_area(
            "💡 Kiến nghị cho công ty"
        )

        if st.form_submit_button(
            "📤 GỬI BÁO CÁO",
            type="primary"
        ):

            execute("""
                INSERT INTO tour_reports
                (
                    tour_id,
                    guide_id,
                    summary,
                    customer_feedback,
                    guide_feedback
                )
                VALUES (?, ?, ?, ?, ?)
            """, (
                tour_id,
                guide[0]["id"],
                summary,
                customer_feedback,
                guide_feedback
            ))

            execute(
                "UPDATE tours SET status='Đã hoàn thành' WHERE id=?",
                (tour_id,)
            )

            st.success(
                "Đã gửi báo cáo tour."
            )


# =========================================================
# REPORTING
# =========================================================

def reports_page():

    st.title("📊 Báo cáo & thống kê")

    tours = query(
        "SELECT * FROM tours",
        fetch=True
    )

    assignments = query(
        "SELECT * FROM assignments",
        fetch=True
    )

    guides = query(
        "SELECT * FROM guides",
        fetch=True
    )

    c1, c2, c3, c4 = st.columns(4)

    c1.metric(
        "🚌 Tổng tour",
        len(tours)
    )

    c2.metric(
        "👨‍💼 HDV",
        len(guides)
    )

    c3.metric(
        "📌 Phân công",
        len(assignments)
    )

    c4.metric(
        "🚨 Sự cố",
        len(query(
            "SELECT * FROM incidents",
            fetch=True
        ))
    )

    st.divider()

    st.subheader(
        "📅 Tour theo trạng thái"
    )

    status_count = {}

    for t in tours:

        status_count[t["status"]] = (
            status_count.get(
                t["status"],
                0
            ) + 1
        )

    for status, count in status_count.items():

        st.write(
            f"{status_badge(status)}: **{count} tour**"
        )

    st.divider()

    st.subheader(
        "👨‍💼 Số tour theo HDV"
    )

    for g in guides:

        count = len([
            a for a in assignments
            if a["guide_id"] == g["id"]
        ])

        st.write(
            f"**{g['full_name']}**: {count} tour"
        )


# =========================================================
# CUSTOMERS PAGE
# =========================================================

def customers_page():

    st.title("👥 Khách đoàn")

    st.info(
        "Phân hệ khách đoàn có thể mở rộng "
        "thành danh sách khách, điểm danh, "
        "phân phòng và quản lý yêu cầu đặc biệt."
    )

    tours = query(
        "SELECT * FROM tours ORDER BY departure_date DESC",
        fetch=True
    )

    if tours:

        data = []

        for t in tours:

            data.append({
                "Mã tour": t["code"],
                "Tên tour": t["name"],
                "Ngày đi": format_date(
                    t["departure_date"]
                ),
                "Ngày về": format_date(
                    t["return_date"]
                ),
                "Số khách": t["customer_count"],
                "Điểm đến": t["destination"],
            })

        st.dataframe(
            data,
            use_container_width=True,
            hide_index=True
        )


# =========================================================
# ROUTER
# =========================================================

def main():

    init_db()

    if not st.session_state.get(
        "logged_in",
        False
    ):

        login_page()
        return

    menu = sidebar()

    if menu == "🏠 Dashboard":
        dashboard()

    elif menu == "📅 Lịch phân công":
        calendar_page()

    elif menu == "📅 Lịch tour của tôi":
        calendar_page()

    elif menu == "🚌 Quản lý tour":
        tour_management()

    elif menu == "🚌 Tour của tôi":
        my_tours()

    elif menu == "👨‍💼 Quản lý HDV":
        guide_management()

    elif menu == "👥 Khách đoàn":
        customers_page()

    elif menu == "🗺️ Lịch trình":
        itinerary_management()

    elif menu == "🔔 Thông báo":
        notifications_page()

    elif menu == "🚨 Sự cố":
        incidents_page()

    elif menu == "📝 Báo cáo tour":
        report_page()

    elif menu == "📊 Báo cáo":
        reports_page()


if __name__ == "__main__":
    main()
