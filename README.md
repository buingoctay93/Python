import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, filedialog
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib import colors
from reportlab.platypus import Table, TableStyle, Image, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
import datetime
import json
import os
import sys
import subprocess
import random
from datetime import datetime, timedelta
import win32print
import win32api
from PIL import Image, ImageTk  # Thêm thư viện để xử lý hình ảnh

class CoffeePOSApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Coffee POS System")
        self.root.geometry("1200x750")
        self.root.withdraw()  # Ẩn cửa sổ chính khi khởi động
        
        # Tài khoản mặc định
        self.users = {
            "admin": {"password": "admin123", "role": "admin"},
            "staff": {"password": "staff123", "role": "staff"}
        }
        
        # Biến lưu trữ
        self.current_user = None
        self.user_role = None
        self.data_file = "coffee_pos_data.json"
        self.drinks = {}
        self.table_orders = {}
        self.table_totals = {}
        self.selected_table = None
        self.inventory = {}
        self.promotions = {
            "Mùa hè": {"discount": 10, "start_date": "01/06/2023", "end_date": "31/08/2023"},
            "Khách hàng thân thiết": {"discount": 15, "start_date": "01/01/2023", "end_date": "31/12/2023"}
        }
        self.payment_methods = ["Tiền mặt", "Chuyển khoản", "Thẻ tín dụng"]
        self.daily_reports = {}
        self.printers = []
        self.selected_printer = None
        self.bill_template = {
            "header": "QUÁN CÀ PHÊ XYZ",
            "address": "123 Đường ABC, Quận 1, TP.HCM",
            "phone": "ĐT: 0123.456.789",
            "footer": "Cảm ơn quý khách! Hẹn gặp lại!",
            "logo": None
        }
        self.max_tables = 12  # Số bàn mặc định
        self.table_colors = {
            "default": "#f8f9fa",
            "selected": "#007bff",
            "occupied": "#ffc107"
        }
        
        # Tạo giao diện đăng nhập
        self.create_login_window()
    
    def create_login_window(self):
        self.login_window = tk.Toplevel(self.root)
        self.login_window.title("POS Py Coffee And Milk Tea")
        self.login_window.geometry("400x300")
        self.login_window.resizable(False, False)
        self.login_window.protocol("WM_DELETE_WINDOW", self.on_close)  # Xử lý khi đóng cửa sổ đăng nhập

        # Thêm hình ảnh
        try:
            image_path = os.path.join(os.path.dirname(__file__), "lycf.png")  # Đường dẫn tương đối đến file ảnh
            image = Image.open(image_path)
            image = image.resize((100, 100), Image.Resampling.LANCZOS)
            photo = ImageTk.PhotoImage(image)
            image_label = tk.Label(self.login_window, image=photo)
            image_label.image = photo  # Lưu tham chiếu để tránh bị xóa
            image_label.pack(pady=10)
        except FileNotFoundError:
            messagebox.showerror("Lỗi", "Không tìm thấy file ảnh 'lycf.png'. Vui lòng kiểm tra lại.")

        # Tiêu đề
        title_label = tk.Label(self.login_window, text="ĐĂNG NHẬP", font=("Arial", 16, "bold"), fg="blue")
        title_label.pack(pady=5)

        # Tên đăng nhập
        username_label = tk.Label(self.login_window, text="User Name", font=("Arial", 10))
        username_label.pack(pady=5)
        self.username_entry = tk.Entry(self.login_window, font=("Arial", 10), width=30)
        self.username_entry.pack(pady=5)

        # Mật khẩu
        password_label = tk.Label(self.login_window, text="Password", font=("Arial", 10))
        password_label.pack(pady=5)
        self.password_entry = tk.Entry(self.login_window, font=("Arial", 10), show="*", width=30)
        self.password_entry.pack(pady=5)

        # Gắn sự kiện Enter vào ô nhập mật khẩu
        self.password_entry.bind("<Return>", lambda event: self.authenticate())

        # Nút đăng nhập
        login_button = tk.Button(self.login_window, text="Đăng nhập", font=("Arial", 12, "bold"), bg="blue", fg="white",
                                 command=self.authenticate)
        login_button.pack(pady=10)
    
    def authenticate(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        
        if username in self.users and self.users[username]["password"] == password:
            self.current_user = username
            self.user_role = self.users[username]["role"]
            self.login_window.destroy()  # Đóng cửa sổ đăng nhập
            self.root.deiconify()  # Hiển thị cửa sổ chính
            self.load_data()
            self.create_main_interface()
        else:
            messagebox.showerror("Lỗi", "Tên đăng nhập hoặc mật khẩu không đúng")
    
    def on_close(self):
        if messagebox.askokcancel("Thoát", "Bạn có chắc chắn muốn thoát ứng dụng?"):
            self.root.destroy()
    
    def load_data(self):
        default_data = {
            "drinks": {
                "Cà phê": [("Cà phê sữa", 20000, None), ("Cà phê đen", 15000, None)],
                "Trà": [("Trà đào", 25000, None), ("Trà vải", 25000, None)],
                "Sinh tố": [("Sinh tố bơ", 30000, None), ("Sinh tố dâu", 35000, None)]
            },
            "table_orders": {i: [] for i in range(1, self.max_tables + 1)},
            "table_totals": {i: 0 for i in range(1, self.max_tables + 1)},
            "inventory": {
                "Cà phê sữa": {"quantity": 100, "unit": "gói"},
                "Cà phê đen": {"quantity": 100, "unit": "gói"},
                "Trà đào": {"quantity": 50, "unit": "gói"},
                "Trà vải": {"quantity": 50, "unit": "gói"},
                "Bơ": {"quantity": 20, "unit": "kg"},
                "Dâu": {"quantity": 15, "unit": "kg"}
            },
            "promotions": {
                "Mùa hè": {"discount": 10, "start_date": "01/06/2023", "end_date": "31/08/2023"},
                "Khách hàng thân thiết": {"discount": 15, "start_date": "01/01/2023", "end_date": "31/12/2023"}
            },
            "daily_reports": {},
            "printers": [],
            "selected_printer": None,
            "bill_template": {
                "header": "QUÁN CÀ PHÊ XYZ",
                "address": "123 Đường ABC, Quận 1, TP.HCM",
                "phone": "ĐT: 0123.456.789",
                "footer": "Cảm ơn quý khách! Hẹn gặp lại!",
                "logo": None
            },
            "max_tables": 12
        }
        
        try:
            if os.path.exists(self.data_file):
                with open(self.data_file, 'r', encoding='utf-8') as f:
                    self.data = json.load(f)
            else:
                self.data = default_data
                
            self.drinks = self.data["drinks"]
            self.table_orders = {int(k): v for k, v in self.data["table_orders"].items()}
            self.table_totals = {int(k): v for k, v in self.data["table_totals"].items()}
            self.inventory = self.data.get("inventory", {})
            self.promotions = self.data.get("promotions", {})
            self.daily_reports = self.data.get("daily_reports", {})
            self.printers = self.data.get("printers", [])
            self.selected_printer = self.data.get("selected_printer")
            self.bill_template = self.data.get("bill_template", default_data["bill_template"])
            self.max_tables = self.data.get("max_tables", 12)
            
            # Lấy danh sách máy in nếu chưa có
            if not self.printers:
                self.get_system_printers()
            
        except Exception as e:
            messagebox.showerror("Lỗi", f"Không thể đọc dữ liệu: {str(e)}")
            self.data = default_data
    
    def save_data(self):
        self.data = {
            "drinks": self.drinks,
            "table_orders": {str(k): v for k, v in self.table_orders.items()},
            "table_totals": {str(k): v for k, v in self.table_totals.items()},
            "inventory": self.inventory,
            "promotions": self.promotions,
            "daily_reports": self.daily_reports,
            "printers": self.printers,
            "selected_printer": self.selected_printer,
            "bill_template": self.bill_template,
            "max_tables": self.max_tables
        }
        
        with open(self.data_file, 'w', encoding='utf-8') as f:
            json.dump(self.data, f, indent=2, ensure_ascii=False)
    
    def get_system_printers(self):
        try:
            self.printers = [printer[2] for printer in win32print.EnumPrinters(2)]
            self.save_data()
        except:
            self.printers = ["Máy in PDF", "Máy in mặc định"]
    
    def create_main_interface(self):
        # Tạo notebook (tab)
        self.notebook = ttk.Notebook(self.root)
        self.notebook.pack(fill=tk.BOTH, expand=True)
        
        # Tab POS
        self.pos_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.pos_tab, text="POS")
        self.create_pos_tab()
        
        # Tab Quản lý
        self.manage_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.manage_tab, text="Quản lý")
        self.create_manage_tab()
        
        # Tab Báo cáo doanh thu
        self.report_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.report_tab, text="Báo cáo doanh thu")
        self.create_report_tab()
        
        # Tab Khuyến mãi
        self.promotion_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.promotion_tab, text="Khuyến mãi")
        self.create_promotion_tab()
        
        # Tab Quản lý kho
        self.inventory_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.inventory_tab, text="Quản lý kho")
        self.create_inventory_tab()
        
        # Tab Cấu hình in
        self.print_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.print_tab, text="Cấu hình in")
        self.create_print_tab()
        
        # Kiểm tra phân quyền
        self.check_permissions()
    
    def check_permissions(self):
        if self.user_role == "staff":
            self.notebook.tab(1, state="disabled")  # Tab Quản lý
            self.notebook.tab(3, state="disabled")  # Tab Khuyến mãi
            self.notebook.tab(4, state="disabled")  # Tab Quản lý kho
            self.notebook.tab(5, state="disabled")  # Tab Cấu hình in
    
    def create_pos_tab(self):
        # Frame chứa danh sách bàn
        table_frame = ttk.LabelFrame(self.pos_tab, text=f"Danh sách bàn (1-{self.max_tables})")
        table_frame.pack(side=tk.LEFT, fill=tk.Y, padx=5, pady=5)

        # Canvas để vẽ bàn
        table_canvas = tk.Canvas(table_frame)
        table_canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=table_canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        table_canvas.configure(yscrollcommand=scrollbar.set)
        scrollable_frame = ttk.Frame(table_canvas)

        scrollable_frame.bind(
            "<Configure>",
            lambda e: table_canvas.configure(scrollregion=table_canvas.bbox("all"))
        )

        table_canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")

        # Tạo danh sách bàn
        self.table_buttons = []
        rows = 10  # Số hàng (10 bàn mỗi cột)
        cols = 3   # Số cột (3 cột)

        for i in range(self.max_tables):
            row = i % rows
            col = i // rows

            # Vẽ hình tròn đại diện cho bàn
            canvas = tk.Canvas(scrollable_frame, width=100, height=100, bg="#f8f9fa", highlightthickness=0)
            canvas.grid(row=row, column=col, padx=10, pady=10)
            canvas.create_oval(10, 10, 90, 90, fill="#D2B48C", outline="#8B4513", width=2)  # Màu gỗ

            # Thêm số bàn vào giữa hình tròn
            canvas.create_text(50, 50, text=f"Bàn {i + 1}", fill="black", font=("Arial", 12, "bold"))

            # Gắn sự kiện chọn bàn
            canvas.bind("<Button-1>", lambda event, table_num=i + 1: self.select_table(table_num))

            self.table_buttons.append(canvas)

        # Frame đồ uống
        drink_frame = ttk.LabelFrame(self.pos_tab, text="Đồ uống")
        drink_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5, pady=5)

        # Nhóm đồ uống
        group_frame = ttk.Frame(drink_frame)
        group_frame.pack(fill=tk.X, pady=5)

        self.group_buttons = {}  # Lưu các nút nhóm để thay đổi màu
        groups = ["Tất cả"] + list(self.drinks.keys())
        for group in groups:
            btn = tk.Button(group_frame, text=group, width=10,
                            command=lambda g=group: self.filter_drinks(g))
            btn.pack(side=tk.LEFT, padx=2)
            self.group_buttons[group] = btn

        # Tìm kiếm đồ uống
        search_frame = ttk.Frame(drink_frame)
        search_frame.pack(fill=tk.X, pady=5)

        tk.Label(search_frame, text="Tìm kiếm:").pack(side=tk.LEFT, padx=5)
        self.search_entry = tk.Entry(search_frame)
        self.search_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=5)
        self.search_entry.bind("<KeyRelease>", lambda event: self.search_drinks())

        # Danh sách đồ uống
        self.drink_canvas = tk.Canvas(drink_frame)
        self.drink_canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = ttk.Scrollbar(drink_frame, orient=tk.VERTICAL, command=self.drink_canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        self.drink_canvas.configure(yscrollcommand=scrollbar.set)
        self.drink_frame = ttk.Frame(self.drink_canvas)
        self.drink_canvas.create_window((0, 0), window=self.drink_frame, anchor="nw")

        self.filter_drinks("Tất cả")

        # Frame danh sách đồ uống đã chọn
        order_frame = ttk.LabelFrame(self.pos_tab, text="Danh sách đồ uống đã chọn")
        order_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5, pady=5)

        self.order_listbox = tk.Listbox(order_frame, height=20)
        self.order_listbox.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

        # Tổng tiền
        self.total_label = tk.Label(order_frame, text="Tổng: 0 đ", font=("Arial", 12, "bold"))
        self.total_label.pack(pady=5)

        # Nút thanh toán & in hóa đơn
        checkout_button = tk.Button(order_frame, text="Thanh toán & In hóa đơn", command=self.checkout)
        checkout_button.pack(pady=10)

        # Frame khuyến mãi
        promotion_frame = ttk.LabelFrame(self.pos_tab, text="Khuyến mãi")
        promotion_frame.pack(side=tk.LEFT, fill=tk.X, padx=5, pady=5)

        tk.Label(promotion_frame, text="Chọn khuyến mãi:").pack(side=tk.LEFT, padx=5)
        self.promotion_combobox = ttk.Combobox(promotion_frame, values=list(self.promotions.keys()), state="readonly")
        self.promotion_combobox.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=5)
        self.promotion_combobox.set("Không áp dụng")  # Giá trị mặc định

        # Frame phương thức thanh toán
        payment_frame = ttk.LabelFrame(self.pos_tab, text="Phương thức thanh toán")
        payment_frame.pack(side=tk.LEFT, fill=tk.X, padx=5, pady=5)

        tk.Label(payment_frame, text="Chọn phương thức:").pack(side=tk.LEFT, padx=5)
        self.payment_method = ttk.Combobox(payment_frame, values=self.payment_methods, state="readonly")
        self.payment_method.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=5)
        self.payment_method.set(self.payment_methods[0])  # Giá trị mặc định
    
    def filter_drinks(self, group):
        for widget in self.drink_frame.winfo_children():
            widget.destroy()
        
        drinks = []
        if group == "Tất cả":
            for group_drinks in self.drinks.values():
                drinks.extend(group_drinks)
        else:
            drinks = self.drinks.get(group, [])
        
        for idx, (name, price, _) in enumerate(drinks):
            tk.Button(self.drink_frame, text=f"{name}\n{price:,} đ", width=15, height=2,
                    command=lambda n=name, p=price: self.add_to_order(n, p)).grid(row=idx//3, column=idx%3, padx=5, pady=5)
        
        self.drink_frame.update_idletasks()
        self.drink_canvas.configure(scrollregion=self.drink_canvas.bbox("all"))
    
    def search_drinks(self):
        query = self.search_entry.get().strip().lower()

        # Xóa các widget cũ trong danh sách đồ uống
        for widget in self.drink_frame.winfo_children():
            widget.destroy()

        # Lọc đồ uống theo từ khóa tìm kiếm
        drinks = []
        for group_drinks in self.drinks.values():
            drinks.extend(group_drinks)

        filtered_drinks = [drink for drink in drinks if query in drink[0].lower()]

        # Hiển thị đồ uống
        for idx, (name, price, _) in enumerate(filtered_drinks):
            tk.Button(self.drink_frame, text=f"{name}\n{price:,} đ", width=15, height=2,
                      command=lambda n=name, p=price: self.add_to_order(n, p)).grid(row=idx // 3, column=idx % 3, padx=5, pady=5)

        self.drink_frame.update_idletasks()
        self.drink_canvas.configure(scrollregion=self.drink_canvas.bbox("all"))
    
    def select_table(self, table_num):
        self.selected_table = table_num
        for i in range(1, self.max_tables + 1):
            self.update_table_button(i)
        self.update_order_listbox()
    
    def update_table_button(self, table_num):
        if table_num > len(self.table_buttons):
            return
            
        total = self.table_totals.get(table_num, 0)
        btn = self.table_buttons[table_num-1]
        btn_text = f"Bàn {table_num}\n{total:,} đ"
        
        if self.selected_table == table_num:
            btn.config(bg=self.table_colors["selected"])
        elif total > 0:
            btn.config(bg=self.table_colors["occupied"])
        else:
            btn.config(bg=self.table_colors["default"])
    
    def add_to_order(self, name, price):
        if not self.selected_table:
            messagebox.showwarning("Cảnh báo", "Vui lòng chọn bàn trước")
            return

        # Kiểm tra tồn kho
        if name in self.inventory and self.inventory[name]["quantity"] <= 0:
            messagebox.showwarning("Hết hàng", f"{name} đã hết hàng trong kho!")
            return

        # Tìm xem món đã có trong đơn chưa
        for i, (item_name, qty, item_price) in enumerate(self.table_orders[self.selected_table]):
            if item_name == name and item_price == price:
                self.table_orders[self.selected_table][i] = (name, qty + 1, price)
                self.table_totals[self.selected_table] += price
                self.update_order_listbox()
                self.update_table_button(self.selected_table)

                # Cập nhật tồn kho
                if name in self.inventory:
                    self.inventory[name]["quantity"] -= 1
                return

        # Thêm món mới
        self.table_orders[self.selected_table].append((name, 1, price))
        self.table_totals[self.selected_table] += price
        self.update_order_listbox()
        self.update_table_button(self.selected_table)

        # Cập nhật tồn kho
        if name in self.inventory:
            self.inventory[name]["quantity"] -= 1
    
    def update_order_listbox(self):
        self.order_listbox.delete(0, tk.END)

        if not self.selected_table or not self.table_orders[self.selected_table]:
            self.total_label.config(text="Tổng: 0 đ")
            return

        total = 0
        for name, qty, price in self.table_orders[self.selected_table]:
            subtotal = qty * price
            total += subtotal
            self.order_listbox.insert(tk.END, f"{name} x{qty} = {subtotal:,} đ")

        self.order_listbox.insert(tk.END, "-" * 40)
        self.order_listbox.insert(tk.END, f"Tổng cộng: {total:,} đ")
        self.total_label.config(text=f"Tổng: {total:,} đ")
    
    def delete_item(self):
        if not self.selected_table:
            return
            
        selection = self.order_listbox.curselection()
        if not selection:
            return
            
        index = selection[0]
        if index >= len(self.table_orders[self.selected_table]):
            return
            
        name, qty, price = self.table_orders[self.selected_table].pop(index)
        self.table_totals[self.selected_table] -= qty * price
        
        # Hoàn trả tồn kho
        if name in self.inventory:
            self.inventory[name]["quantity"] += qty
        
        self.update_order_listbox()
        self.update_table_button(self.selected_table)
    
    def checkout(self):
        if not self.selected_table or not self.table_orders[self.selected_table]:
            messagebox.showwarning("Cảnh báo", "Không có món nào để thanh toán")
            return

        total = self.table_totals[self.selected_table]

        # Áp dụng khuyến mãi
        promotion = self.promotion_combobox.get()
        discount = 0
        if promotion and promotion in self.promotions:
            discount = self.promotions[promotion]["discount"]
            total = total * (100 - discount) / 100
            messagebox.showinfo("Khuyến mãi", f"Áp dụng khuyến mãi {promotion}: giảm {discount}%")

        # Lấy phương thức thanh toán
        payment_method = self.payment_method.get()
        if not payment_method:
            messagebox.showwarning("Cảnh báo", "Vui lòng chọn phương thức thanh toán")
            return

        # Lưu báo cáo doanh thu
        today = datetime.now().strftime("%d/%m/%Y")
        if today not in self.daily_reports:
            self.daily_reports[today] = {"total": 0, "orders": 0, "payment_methods": {pm: 0 for pm in self.payment_methods}}

        self.daily_reports[today]["total"] += total
        self.daily_reports[today]["orders"] += 1
        self.daily_reports[today]["payment_methods"][payment_method] += total

        if messagebox.askyesno("Xác nhận", 
                             f"Thanh toán bàn {self.selected_table}\n"
                             f"Phương thức: {payment_method}\n"
                             f"Khuyến mãi: {promotion if promotion else 'Không'}\n"
                             f"Tổng: {total:,.0f} đ"):
            self.table_orders[self.selected_table] = []
            self.table_totals[self.selected_table] = 0
            self.update_order_listbox()
            self.update_table_button(self.selected_table)
            self.save_data()

            # In hóa đơn tự động
            self.print_bill()
    
    def print_bill(self):
        print(f"Table orders: {self.table_orders[self.selected_table]}")  # Debug: Kiểm tra danh sách món
        if not self.selected_table:
            messagebox.showwarning("Cảnh báo", "Vui lòng chọn bàn trước khi in hóa đơn")
            return

        if not self.table_orders[self.selected_table]:
            messagebox.showwarning("Cảnh báo", "Bàn này chưa có món nào để in hóa đơn")
            return

        # ... tiếp tục xử lý in hóa đơn ...
    
    def edit_bill(self):
        if not self.selected_table or not self.table_orders[self.selected_table]:
            messagebox.showwarning("Cảnh báo", "Không có hóa đơn để chỉnh sửa")
            return
        
        edit_window = tk.Toplevel(self.root)
        edit_window.title(f"Chỉnh sửa hóa đơn bàn {self.selected_table}")
        edit_window.geometry("500x400")
        
        # Danh sách món đã order
        tk.Label(edit_window, text="Danh sách món:", font=("Arial", 12, "bold")).pack(pady=5)
        
        list_frame = tk.Frame(edit_window)
        list_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        
        scrollbar = tk.Scrollbar(list_frame)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        self.edit_listbox = tk.Listbox(list_frame, yscrollcommand=scrollbar.set, width=50)
        self.edit_listbox.pack(fill=tk.BOTH, expand=True)
        
        scrollbar.config(command=self.edit_listbox.yview)
        
        for item in self.table_orders[self.selected_table]:
            name, qty, price = item
            self.edit_listbox.insert(tk.END, f"{name} x{qty} = {qty*price:,} đ")
        
        # Nút chức năng
        btn_frame = tk.Frame(edit_window)
        btn_frame.pack(pady=10)
        
        tk.Button(btn_frame, text="Sửa số lượng", command=self.edit_quantity).pack(side=tk.LEFT, padx=5)
        tk.Button(btn_frame, text="Xóa món", command=self.delete_item_from_edit).pack(side=tk.LEFT, padx=5)
        tk.Button(btn_frame, text="Lưu thay đổi", command=lambda: self.save_edit(edit_window)).pack(side=tk.LEFT, padx=5)
    
    def edit_quantity(self):
        selection = self.edit_listbox.curselection()
        if not selection:
            return
            
        index = selection[0]
        if index >= len(self.table_orders[self.selected_table]):
            return
            
        name, old_qty, price = self.table_orders[self.selected_table][index]
        
        new_qty = simpledialog.askinteger("Sửa số lượng", f"Nhập số lượng mới cho {name}:", 
                                        initialvalue=old_qty, minvalue=1)
        if new_qty and new_qty != old_qty:
            # Cập nhật số lượng và tổng tiền
            diff = new_qty - old_qty
            self.table_orders[self.selected_table][index] = (name, new_qty, price)
            self.table_totals[self.selected_table] += diff * price
            
            # Cập nhật tồn kho
            if name in self.inventory:
                self.inventory[name]["quantity"] -= diff
            
            # Cập nhật listbox
            self.edit_listbox.delete(index)
            self.edit_listbox.insert(index, f"{name} x{new_qty} = {new_qty*price:,} đ")
    
    def delete_item_from_edit(self):
        selection = self.edit_listbox.curselection()
        if not selection:
            return
            
        index = selection[0]
        if index >= len(self.table_orders[self.selected_table]):
            return
            
        name, qty, price = self.table_orders[self.selected_table].pop(index)
        self.table_totals[self.selected_table] -= qty * price
        
        # Hoàn trả tồn kho
        if name in self.inventory:
            self.inventory[name]["quantity"] += qty
        
        # Cập nhật listbox
        self.edit_listbox.delete(index)
    
    def save_edit(self, window):
        self.update_order_listbox()
        self.update_table_button(self.selected_table)
        window.destroy()
        messagebox.showinfo("Thành công", "Đã cập nhật hóa đơn!")
    
    def create_manage_tab(self):
        # Frame thêm món
        add_frame = ttk.LabelFrame(self.manage_tab, text="Thêm món mới")
        add_frame.pack(fill=tk.X, padx=5, pady=5)

        tk.Label(add_frame, text="Tên món:").grid(row=0, column=0, padx=5, pady=5)
        self.new_drink_name = tk.Entry(add_frame)
        self.new_drink_name.grid(row=0, column=1, padx=5, pady=5)

        tk.Label(add_frame, text="Giá:").grid(row=1, column=0, padx=5, pady=5)
        self.new_drink_price = tk.Entry(add_frame)
        self.new_drink_price.grid(row=1, column=1, padx=5, pady=5)

        tk.Label(add_frame, text="Nhóm:").grid(row=2, column=0, padx=5, pady=5)
        self.new_drink_group = ttk.Combobox(add_frame, values=list(self.drinks.keys()))
        self.new_drink_group.grid(row=2, column=1, padx=5, pady=5)

        tk.Button(add_frame, text="Thêm món", command=self.add_new_drink).grid(row=3, column=1, pady=5)

        # Frame tạo nhóm mới
        group_frame = ttk.LabelFrame(self.manage_tab, text="Tạo nhóm mới")
        group_frame.pack(fill=tk.X, padx=5, pady=5)

        tk.Label(group_frame, text="Tên nhóm:").grid(row=0, column=0, padx=5, pady=5)
        self.new_group_name = tk.Entry(group_frame)
        self.new_group_name.grid(row=0, column=1, padx=5, pady=5)

        tk.Button(group_frame, text="Tạo nhóm", command=self.add_new_group).grid(row=0, column=2, padx=5, pady=5)

        # Frame quản lý bàn
        table_frame = ttk.LabelFrame(self.manage_tab, text="Quản lý bàn")
        table_frame.pack(fill=tk.X, padx=5, pady=5)

        tk.Label(table_frame, text="Số lượng bàn:").grid(row=0, column=0, padx=5, pady=5)
        self.table_count = tk.Entry(table_frame)
        self.table_count.insert(0, str(self.max_tables))
        self.table_count.grid(row=0, column=1, padx=5, pady=5)

        tk.Button(table_frame, text="Cập nhật số bàn", command=self.update_table_count).grid(row=0, column=2, padx=5, pady=5)

        # Frame quản lý người dùng (chỉ admin)
        if self.user_role == "admin":
            user_frame = ttk.LabelFrame(self.manage_tab, text="Quản lý người dùng")
            user_frame.pack(fill=tk.X, padx=5, pady=5)

            tk.Button(user_frame, text="Thêm người dùng", command=self.add_user).pack(side=tk.LEFT, padx=5, pady=5)
            tk.Button(user_frame, text="Đổi mật khẩu", command=self.change_password).pack(side=tk.LEFT, padx=5, pady=5)
    
    def add_new_group(self):
        group_name = self.new_group_name.get().strip()

        if not group_name:
            messagebox.showwarning("Cảnh báo", "Vui lòng nhập tên nhóm")
            return

        if group_name in self.drinks:
            messagebox.showerror("Lỗi", "Nhóm đã tồn tại")
            return

        # Thêm nhóm mới vào danh sách
        self.drinks[group_name] = []
        self.save_data()

        # Cập nhật danh sách nhóm trong combobox
        self.new_drink_group["values"] = list(self.drinks.keys())

        messagebox.showinfo("Thành công", f"Đã tạo nhóm '{group_name}'")
        self.new_group_name.delete(0, tk.END)
    
    def update_table_count(self):
        try:
            new_count = int(self.table_count.get())
            if new_count < 1 or new_count > 50:
                raise ValueError
        except ValueError:
            messagebox.showerror("Lỗi", "Số lượng bàn phải là số từ 1 đến 50")
            return

        # Cập nhật dữ liệu bàn
        old_count = self.max_tables
        self.max_tables = new_count

        # Thêm bàn mới nếu cần
        for i in range(old_count + 1, new_count + 1):
            if i not in self.table_orders:
                self.table_orders[i] = []
                self.table_totals[i] = 0

        # Xóa bàn thừa nếu cần
        for i in range(new_count + 1, old_count + 1):
            if i in self.table_orders:
                del self.table_orders[i]
                del self.table_totals[i]

        # Xóa nội dung cũ trong self.pos_tab
        for widget in self.pos_tab.winfo_children():
            widget.destroy()

        # Tạo lại giao diện tab POS
        self.create_pos_tab()

        # Lưu dữ liệu và hiển thị thông báo
        self.save_data()
        messagebox.showinfo("Thành công", f"Đã cập nhật số lượng bàn thành {new_count}")
    
    def add_new_drink(self):
        name = self.new_drink_name.get().strip()
        price_str = self.new_drink_price.get().strip()
        group = self.new_drink_group.get().strip()
        
        if not name or not price_str or not group:
            messagebox.showwarning("Cảnh báo", "Vui lòng nhập đầy đủ thông tin")
            return
            
        try:
            price = int(price_str)
        except ValueError:
            messagebox.showerror("Lỗi", "Giá phải là số nguyên")
            return
            
        if group not in self.drinks:
            self.drinks[group] = []
            
        self.drinks[group].append([name, price, None])
        self.save_data()
        
        messagebox.showinfo("Thành công", "Đã thêm món mới")
        self.new_drink_name.delete(0, tk.END)
        self.new_drink_price.delete(0, tk.END)
    
    def add_user(self):
        if self.user_role != "admin":
            messagebox.showerror("Lỗi", "Bạn không có quyền thực hiện chức năng này")
            return
            
        add_window = tk.Toplevel(self.root)
        add_window.title("Thêm người dùng")
        add_window.geometry("300x200")
        
        tk.Label(add_window, text="Tên đăng nhập:").pack(pady=5)
        username_entry = tk.Entry(add_window)
        username_entry.pack(pady=5)
        
        tk.Label(add_window, text="Mật khẩu:").pack(pady=5)
        password_entry = tk.Entry(add_window, show="*")
        password_entry.pack(pady=5)
        
        tk.Label(add_window, text="Vai trò:").pack(pady=5)
        role_var = tk.StringVar(value="staff")
        tk.Radiobutton(add_window, text="Nhân viên", variable=role_var, value="staff").pack()
        tk.Radiobutton(add_window, text="Quản lý", variable=role_var, value="admin").pack()
        
        def save_user():
            username = username_entry.get().strip()
            password = password_entry.get().strip()
            role = role_var.get()
            
            if not username or not password:
                messagebox.showwarning("Cảnh báo", "Vui lòng nhập đầy đủ thông tin")
                return
                
            if username in self.users:
                messagebox.showerror("Lỗi", "Tên đăng nhập đã tồn tại")
                return
                
            self.users[username] = {"password": password, "role": role}
            messagebox.showinfo("Thành công", "Đã thêm người dùng mới")
            add_window.destroy()
        
        tk.Button(add_window, text="Lưu", command=save_user).pack(pady=10)
    
    def change_password(self):
        if self.user_role != "admin":
            messagebox.showerror("Lỗi", "Bạn không có quyền thực hiện chức năng này")
            return
            
        change_window = tk.Toplevel(self.root)
        change_window.title("Đổi mật khẩu")
        change_window.geometry("300x150")
        
        tk.Label(change_window, text="Tên đăng nhập:").pack(pady=5)
        username_entry = tk.Entry(change_window)
        username_entry.pack(pady=5)
        
        tk.Label(change_window, text="Mật khẩu mới:").pack(pady=5)
        password_entry = tk.Entry(change_window, show="*")
        password_entry.pack(pady=5)
        
        def save_password():
            username = username_entry.get().strip()
            password = password_entry.get().strip()
            
            if not username or not password:
                messagebox.showwarning("Cảnh báo", "Vui lòng nhập đầy đủ thông tin")
                return
                
            if username not in self.users:
                messagebox.showerror("Lỗi", "Tên đăng nhập không tồn tại")
                return
                
            self.users[username]["password"] = password
            messagebox.showinfo("Thành công", "Đã đổi mật khẩu")
            change_window.destroy()
        
        tk.Button(change_window, text="Lưu", command=save_password).pack(pady=10)
    
    def create_report_tab(self):
        # Frame lựa chọn thời gian
        time_frame = ttk.LabelFrame(self.report_tab, text="Lọc theo thời gian")
        time_frame.pack(fill=tk.X, padx=5, pady=5)
        
        tk.Label(time_frame, text="Từ ngày:").grid(row=0, column=0, padx=5, pady=5)
        self.from_date = tk.Entry(time_frame)
        self.from_date.grid(row=0, column=1, padx=5, pady=5)
        
        tk.Label(time_frame, text="Đến ngày:").grid(row=0, column=2, padx=5, pady=5)
        self.to_date = tk.Entry(time_frame)
        self.to_date.grid(row=0, column=3, padx=5, pady=5)
        
        tk.Button(time_frame, text="Xem báo cáo", command=self.generate_report).grid(row=0, column=4, padx=5, pady=5)
        
        # Frame kết quả báo cáo
        result_frame = ttk.LabelFrame(self.report_tab, text="Kết quả")
        result_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        self.report_text = tk.Text(result_frame, wrap=tk.WORD)
        self.report_text.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Nút xuất báo cáo
        tk.Button(self.report_tab, text="Xuất báo cáo PDF", command=self.export_report).pack(pady=5)
    
    def generate_report(self):
        from_date_str = self.from_date.get().strip()
        to_date_str = self.to_date.get().strip()
        
        try:
            from_date = datetime.strptime(from_date_str, "%d/%m/%Y") if from_date_str else None
            to_date = datetime.strptime(to_date_str, "%d/%m/%Y") if to_date_str else None
        except ValueError:
            messagebox.showerror("Lỗi", "Định dạng ngày không hợp lệ. Vui lòng nhập dd/mm/yyyy")
            return
        
        report_data = []
        total_revenue = 0
        total_orders = 0
        
        for date_str, data in self.daily_reports.items():
            try:
                date = datetime.strptime(date_str, "%d/%m/%Y")
            except ValueError:
                continue
                
            if (from_date is None or date >= from_date) and (to_date is None or date <= to_date):
                report_data.append((date, data))
                total_revenue += data["total"]
                total_orders += data["orders"]
        
        # Sắp xếp theo ngày
        report_data.sort()
        
        # Hiển thị kết quả
        self.report_text.delete(1.0, tk.END)
        
        if not report_data:
            self.report_text.insert(tk.END, "Không có dữ liệu trong khoảng thời gian đã chọn")
            return
        
        self.report_text.insert(tk.END, f"BÁO CÁO DOANH THU TỪ {from_date_str if from_date_str else 'đầu'} ĐẾN {to_date_str if to_date_str else 'cuối'}\n\n")
        self.report_text.insert(tk.END, f"{'Ngày':<15}{'Số đơn':<10}{'Doanh thu':<15}{'Phương thức thanh toán'}\n")
        self.report_text.insert(tk.END, "-"*60 + "\n")
        
        for date, data in report_data:
            date_str = date.strftime("%d/%m/%Y")
            payment_methods = ", ".join([f"{pm}: {amt:,.0f} đ" for pm, amt in data["payment_methods"].items() if amt > 0])
            self.report_text.insert(tk.END, f"{date_str:<15}{data['orders']:<10}{data['total']:,.0f} đ{'':<5}{payment_methods}\n")
        
        self.report_text.insert(tk.END, "-"*60 + "\n")
        self.report_text.insert(tk.END, f"{'Tổng cộng':<15}{total_orders:<10}{total_revenue:,.0f} đ\n")
    
    def export_report(self):
        if not self.report_text.get(1.0, tk.END).strip():
            messagebox.showwarning("Cảnh báo", "Không có dữ liệu để xuất báo cáo")
            return
            
        filename = f"bao_cao_doanh_thu_{datetime.now().strftime('%Y%m%d_%H%M%S')}.pdf"
        c = canvas.Canvas(filename, pagesize=A4)
        width, height = A4
        
        # Header
        c.setFont("Helvetica-Bold", 16)
        c.drawCentredString(width/2, height-50, "BÁO CÁO DOANH THU")
        c.setFont("Helvetica", 10)
        c.drawString(50, height-80, f"Ngày xuất báo cáo: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
        c.drawString(50, height-95, f"Nhân viên: {self.current_user}")
        
        # Lấy nội dung từ text widget
        content = self.report_text.get(1.0, tk.END)
        lines = content.split("\n")
        
        # Vẽ nội dung
        y = height-120
        for line in lines:
            if y < 50:  # Hết trang
                c.showPage()
                y = height-50
                c.setFont("Helvetica", 10)
            
            c.drawString(50, y, line)
            y -= 15
        
        c.save()
        messagebox.showinfo("Thành công", f"Đã xuất báo cáo thành {filename}")
        
        # Mở file PDF
        if os.name == 'nt':  # Windows
            os.startfile(filename)
        else:  # Mac/Linux
            opener = "open" if sys.platform == "darwin" else "xdg-open"
            subprocess.call([opener, filename])
    
    def create_promotion_tab(self):
        # Frame thêm khuyến mãi
        add_frame = ttk.LabelFrame(self.promotion_tab, text="Thêm khuyến mãi mới")
        add_frame.pack(fill=tk.X, padx=5, pady=5)
        
        tk.Label(add_frame, text="Tên khuyến mãi:").grid(row=0, column=0, padx=5, pady=5)
        self.new_promo_name = tk.Entry(add_frame)
        self.new_promo_name.grid(row=0, column=1, padx=5, pady=5)
        
        tk.Label(add_frame, text="Giảm giá (%):").grid(row=1, column=0, padx=5, pady=5)
        self.new_promo_discount = tk.Entry(add_frame)
        self.new_promo_discount.grid(row=1, column=1, padx=5, pady=5)
        
        tk.Label(add_frame, text="Từ ngày:").grid(row=2, column=0, padx=5, pady=5)
        self.new_promo_start = tk.Entry(add_frame)
        self.new_promo_start.grid(row=2, column=1, padx=5, pady=5)
        
        tk.Label(add_frame, text="Đến ngày:").grid(row=3, column=0, padx=5, pady=5)
        self.new_promo_end = tk.Entry(add_frame)
        self.new_promo_end.grid(row=3, column=1, padx=5, pady=5)
        
        tk.Button(add_frame, text="Thêm khuyến mãi", command=self.add_new_promotion).grid(row=4, column=1, pady=5)
        
        # Frame danh sách khuyến mãi
        list_frame = ttk.LabelFrame(self.promotion_tab, text="Danh sách khuyến mãi")
        list_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        columns = ("Tên", "Giảm giá", "Từ ngày", "Đến ngày", "Trạng thái")
        self.promo_tree = ttk.Treeview(list_frame, columns=columns, show="headings")
        
        for col in columns:
            self.promo_tree.heading(col, text=col)
            self.promo_tree.column(col, width=100, anchor="center")
        
        self.promo_tree.pack(fill=tk.BOTH, expand=True)
        
        # Nút xóa khuyến mãi
        tk.Button(list_frame, text="Xóa khuyến mãi", command=self.delete_promotion).pack(pady=5)
        
        # Cập nhật danh sách
        self.update_promotion_list()
    
    def add_new_promotion(self):
        name = self.new_promo_name.get().strip()
        discount_str = self.new_promo_discount.get().strip()
        start_date = self.new_promo_start.get().strip()
        end_date = self.new_promo_end.get().strip()
        
        if not name or not discount_str or not start_date or not end_date:
            messagebox.showwarning("Cảnh báo", "Vui lòng nhập đầy đủ thông tin")
            return
            
        try:
            discount = int(discount_str)
            if discount <= 0 or discount > 100:
                raise ValueError
        except ValueError:
            messagebox.showerror("Lỗi", "Giảm giá phải là số từ 1 đến 100")
            return
            
        try:
            datetime.strptime(start_date, "%d/%m/%Y")
            datetime.strptime(end_date, "%d/%m/%Y")
        except ValueError:
            messagebox.showerror("Lỗi", "Định dạng ngày không hợp lệ. Vui lòng nhập dd/mm/yyyy")
            return
            
        self.promotions[name] = {
            "discount": discount,
            "start_date": start_date,
            "end_date": end_date
        }
        
        self.save_data()
        self.update_promotion_list()
        
        messagebox.showinfo("Thành công", "Đã thêm khuyến mãi mới")
        self.new_promo_name.delete(0, tk.END)
        self.new_promo_discount.delete(0, tk.END)
        self.new_promo_start.delete(0, tk.END)
        self.new_promo_end.delete(0, tk.END)
    
    def update_promotion_list(self):
        for item in self.promo_tree.get_children():
            self.promo_tree.delete(item)
            
        today = datetime.now().date()
        
        for name, promo in self.promotions.items():
            try:
                start_date = datetime.strptime(promo["start_date"], "%d/%m/%Y").date()
                end_date = datetime.strptime(promo["end_date"], "%d/%m/%Y").date()
                
                if today < start_date:
                    status = "Chưa bắt đầu"
                elif today > end_date:
                    status = "Đã hết hạn"
                else:
                    status = "Đang áp dụng"
                    
                self.promo_tree.insert("", tk.END, values=(
                    name,
                    f"{promo['discount']}%",
                    promo["start_date"],
                    promo["end_date"],
                    status
                ))
            except:
                continue
    
    def delete_promotion(self):
        selection = self.promo_tree.selection()
        if not selection:
            return
            
        item = self.promo_tree.item(selection[0])
        promo_name = item["values"][0]
        
        if messagebox.askyesno("Xác nhận", f"Bạn có chắc muốn xóa khuyến mãi '{promo_name}'?"):
            del self.promotions[promo_name]
            self.save_data()
            self.update_promotion_list()
            messagebox.showinfo("Thành công", "Đã xóa khuyến mãi")
    
    def create_inventory_tab(self):
        # Frame thêm hàng vào kho
        add_frame = ttk.LabelFrame(self.inventory_tab, text="Thêm hàng vào kho")
        add_frame.pack(fill=tk.X, padx=5, pady=5)
        
        tk.Label(add_frame, text="Tên hàng:").grid(row=0, column=0, padx=5, pady=5)
        self.new_item_name = tk.Entry(add_frame)
        self.new_item_name.grid(row=0, column=1, padx=5, pady=5)
        
        tk.Label(add_frame, text="Số lượng:").grid(row=1, column=0, padx=5, pady=5)
        self.new_item_quantity = tk.Entry(add_frame)
        self.new_item_quantity.grid(row=1, column=1, padx=5, pady=5)
        
        tk.Label(add_frame, text="Đơn vị:").grid(row=2, column=0, padx=5, pady=5)
        self.new_item_unit = ttk.Combobox(add_frame, values=["gói", "kg", "lít", "chai", "lon"])
        self.new_item_unit.grid(row=2, column=1, padx=5, pady=5)
        self.new_item_unit.current(0)
        
        tk.Button(add_frame, text="Thêm hàng", command=self.add_inventory_item).grid(row=3, column=1, pady=5)
        
        # Frame danh sách hàng trong kho
        list_frame = ttk.LabelFrame(self.inventory_tab, text="Danh sách hàng trong kho")
        list_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        columns = ("Tên hàng", "Số lượng", "Đơn vị", "Trạng thái")
        self.inventory_tree = ttk.Treeview(list_frame, columns=columns, show="headings")
        
        for col in columns:
            self.inventory_tree.heading(col, text=col)
            self.inventory_tree.column(col, width=100, anchor="center")
        
        self.inventory_tree.pack(fill=tk.BOTH, expand=True)
        
        # Nút cập nhật số lượng
        tk.Button(list_frame, text="Cập nhật số lượng", command=self.update_inventory_item).pack(pady=5)
        
        # Cập nhật danh sách
        self.update_inventory_list()
    
    def add_inventory_item(self):
        name = self.new_item_name.get().strip()
        quantity_str = self.new_item_quantity.get().strip()
        unit = self.new_item_unit.get().strip()
        
        if not name or not quantity_str or not unit:
            messagebox.showwarning("Cảnh báo", "Vui lòng nhập đầy đủ thông tin")
            return
            
        try:
            quantity = int(quantity_str)
            if quantity <= 0:
                raise ValueError
        except ValueError:
            messagebox.showerror("Lỗi", "Số lượng phải là số nguyên dương")
            return
        
        if name in self.inventory:
            messagebox.showwarning("Cảnh báo", "Hàng đã tồn tại trong kho, vui lòng cập nhật số lượng")
            return
        
        self.inventory[name] = {
            "quantity": quantity,
            "unit": unit
        }
        self.save_data()
        self.update_inventory_list()
        messagebox.showinfo("Thành công", "Đã thêm hàng vào kho")
        self.new_item_name.delete(0, tk.END)
        self.new_item_quantity.delete(0, tk.END)
    
    def update_inventory_item(self):
        selection = self.inventory_tree.selection()
        if not selection:
            return
            
        item = self.inventory_tree.item(selection[0])
        item_name = item["values"][0]
        
        new_quantity = simpledialog.askinteger("Cập nhật số lượng", 
                                             f"Nhập số lượng mới cho {item_name}:",
                                             minvalue=0)
        if new_quantity is not None:
            self.inventory[item_name]["quantity"] = new_quantity
            self.save_data()
            self.update_inventory_list()
            messagebox.showinfo("Thành công", "Đã cập nhật số lượng")
    
    def update_inventory_list(self):
        for item in self.inventory_tree.get_children():
            self.inventory_tree.delete(item)
            
        for name, data in self.inventory.items():
            quantity = data["quantity"]
            unit = data["unit"]
            
            if quantity <= 0:
                status = "Hết hàng"
            elif quantity < 10:
                status = "Sắp hết"
            else:
                status = "Còn hàng"
                
            self.inventory_tree.insert("", tk.END, values=(
                name,
                quantity,
                unit,
                status
            ))
    
    def create_print_tab(self):
        # Frame chọn máy in
        printer_frame = ttk.LabelFrame(self.print_tab, text="Chọn máy in")
        printer_frame.pack(fill=tk.X, padx=5, pady=5)
        
        tk.Label(printer_frame, text="Máy in:").grid(row=0, column=0, padx=5, pady=5)
        self.printer_combobox = ttk.Combobox(printer_frame, values=self.printers, state="readonly")
        self.printer_combobox.grid(row=0, column=1, padx=5, pady=5)
        
        if self.selected_printer and self.selected_printer in self.printers:
            self.printer_combobox.set(self.selected_printer)
        
        tk.Button(printer_frame, text="Lưu cài đặt", command=self.save_printer_setting).grid(row=0, column=2, padx=5, pady=5)
        
        # Frame cấu hình hóa đơn
        template_frame = ttk.LabelFrame(self.print_tab, text="Mẫu hóa đơn")
        template_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Thông tin cửa hàng
        tk.Label(template_frame, text="Tên cửa hàng:").grid(row=0, column=0, padx=5, pady=5, sticky="e")
        self.shop_name = tk.Entry(template_frame)
        self.shop_name.insert(0, self.bill_template["header"])
        self.shop_name.grid(row=0, column=1, padx=5, pady=5, sticky="ew")
        
        tk.Label(template_frame, text="Địa chỉ:").grid(row=1, column=0, padx=5, pady=5, sticky="e")
        self.shop_address = tk.Entry(template_frame)
        self.shop_address.insert(0, self.bill_template["address"])
        self.shop_address.grid(row=1, column=1, padx=5, pady=5, sticky="ew")
        
        tk.Label(template_frame, text="Điện thoại:").grid(row=2, column=0, padx=5, pady=5, sticky="e")
        self.shop_phone = tk.Entry(template_frame)
        self.shop_phone.insert(0, self.bill_template["phone"])
        self.shop_phone.grid(row=2, column=1, padx=5, pady=5, sticky="ew")
        
        tk.Label(template_frame, text="Lời cảm ơn:").grid(row=3, column=0, padx=5, pady=5, sticky="e")
        self.shop_footer = tk.Entry(template_frame)
        self.shop_footer.insert(0, self.bill_template["footer"])
        self.shop_footer.grid(row=3, column=1, padx=5, pady=5, sticky="ew")
        
        # Logo cửa hàng
        tk.Label(template_frame, text="Logo:").grid(row=4, column=0, padx=5, pady=5, sticky="e")
        self.logo_path = tk.StringVar()
        self.logo_path.set(self.bill_template["logo"] if self.bill_template["logo"] else "")
        
        logo_frame = tk.Frame(template_frame)
        logo_frame.grid(row=4, column=1, padx=5, pady=5, sticky="ew")
        
        self.logo_entry = tk.Entry(logo_frame, textvariable=self.logo_path, state="readonly")
        self.logo_entry.pack(side=tk.LEFT, fill=tk.X, expand=True)
        
        tk.Button(logo_frame, text="Chọn...", command=self.select_logo).pack(side=tk.LEFT, padx=5)
        
        # Nút lưu cấu hình
        tk.Button(template_frame, text="Lưu mẫu hóa đơn", command=self.save_bill_template).grid(row=5, column=1, pady=10)
    
    def save_printer_setting(self):
        printer = self.printer_combobox.get()
        if printer:
            self.selected_printer = printer
            self.save_data()
            messagebox.showinfo("Thành công", f"Đã chọn máy in: {printer}")
    
    def select_logo(self):
        filename = filedialog.askopenfilename(
            title="Chọn logo", 
            filetypes=(("Image files", "*.png;*.jpg;*.jpeg"), ("All files", "*.*"))
        )
        if filename:
            self.logo_path.set(filename)
    
    def save_bill_template(self):
        self.bill_template = {
            "header": self.shop_name.get(),
            "address": self.shop_address.get(),
            "phone": self.shop_phone.get(),
            "footer": self.shop_footer.get(),
            "logo": self.logo_path.get()
        }
        self.save_data()
        messagebox.showinfo("Thành công", "Đã lưu mẫu hóa đơn")

if __name__ == "__main__":
    import sys
    try:
        # Khởi tạo ứng dụng
        app = tk.Tk()
        app.title("Coffee POS System")
        pos_app = CoffeePOSApp(app)
        
        # Xử lý sự kiện đóng ứng dụng
        def on_closing():
            if messagebox.askokcancel("Thoát", "Bạn có chắc chắn muốn thoát ứng dụng?"):
                pos_app.save_data()  # Lưu dữ liệu trước khi thoát
                app.destroy()
        
        app.protocol("WM_DELETE_WINDOW", on_closing)
        
        # Chạy ứng dụng
        app.mainloop()
    except Exception as e:
        print(f"Lỗi xảy ra: {e}")
        sys.exit(1)
