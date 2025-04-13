
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

class CoffeePOSApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Coffee POS System")
        self.root.geometry("1200x750")
        self.root.withdraw()  # Hide main window until login
        
        # Set theme colors
        self.bg_color = "#f5f5f5"
        self.primary_color = "#4e73df"
        self.secondary_color = "#858796"
        self.success_color = "#1cc88a"
        self.danger_color = "#e74a3b"
        self.warning_color = "#f6c23e"
        self.text_color = "#5a5c69"
        
        # Configure styles
        self.style = ttk.Style()
        self.style.theme_use('clam')
        
        # Configure styles for widgets
        self.style.configure('TFrame', background=self.bg_color)
        self.style.configure('TLabel', background=self.bg_color, foreground=self.text_color)
        self.style.configure('TButton', font=('Arial', 10), padding=5)
        self.style.configure('Primary.TButton', foreground='white', background=self.primary_color)
        self.style.configure('Success.TButton', foreground='white', background=self.success_color)
        self.style.configure('Danger.TButton', foreground='white', background=self.danger_color)
        self.style.configure('Warning.TButton', foreground='white', background=self.warning_color)
        self.style.map('Primary.TButton', 
                      background=[('active', '#3a56b5'), ('pressed', '#2e4372')])
        self.style.map('Success.TButton', 
                      background=[('active', '#17a673'), ('pressed', '#0f7a56')])
        self.style.map('Danger.TButton', 
                      background=[('active', '#be2617'), ('pressed', '#8c1d11')])
        self.style.map('Warning.TButton', 
                      background=[('active', '#d4a20c'), ('pressed', '#a17a09')])
        
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
        self.promotions = {}
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
            "selected": self.primary_color,
            "occupied": self.warning_color
        }
        
        # Tạo giao diện đăng nhập
        self.create_login_window()
        
    def create_login_window(self):
        self.login_window = tk.Toplevel(self.root)
        self.login_window.title("Đăng nhập")
        self.login_window.geometry("400x300")
        self.login_window.resizable(False, False)
        self.login_window.protocol("WM_DELETE_WINDOW", self.on_close)
        
        # Center the login window
        window_width = 400
        window_height = 300
        screen_width = self.login_window.winfo_screenwidth()
        screen_height = self.login_window.winfo_screenheight()
        x = (screen_width - window_width) // 2
        y = (screen_height - window_height) // 2
        self.login_window.geometry(f"{window_width}x{window_height}+{x}+{y}")
        
        # Styling
        login_frame = ttk.Frame(self.login_window, style='TFrame')
        login_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)
        
        # Header
        header = ttk.Label(login_frame, text="COFFEE POS SYSTEM", 
                          font=('Arial', 16, 'bold'), foreground=self.primary_color)
        header.pack(pady=(0, 20))
        
        # Username
        username_frame = ttk.Frame(login_frame)
        username_frame.pack(fill=tk.X, pady=5)
        ttk.Label(username_frame, text="Tên đăng nhập:").pack(side=tk.LEFT, padx=(0, 10))
        self.username_entry = ttk.Entry(username_frame)
        self.username_entry.pack(fill=tk.X, expand=True)
        
        # Password
        password_frame = ttk.Frame(login_frame)
        password_frame.pack(fill=tk.X, pady=5)
        ttk.Label(password_frame, text="Mật khẩu:").pack(side=tk.LEFT, padx=(0, 10))
        self.password_entry = ttk.Entry(password_frame, show="*")
        self.password_entry.pack(fill=tk.X, expand=True)
        
        # Login button
        btn_frame = ttk.Frame(login_frame)
        btn_frame.pack(pady=20)
        ttk.Button(btn_frame, text="Đăng nhập", style='Primary.TButton', 
                  command=self.authenticate).pack(fill=tk.X, padx=50, ipady=5)
        
        # Bind Enter key to login
        self.password_entry.bind('<Return>', lambda event: self.authenticate())
        
        # Focus on username field
        self.username_entry.focus_set()
    
    def on_close(self):
        self.root.destroy()
    
    def authenticate(self):
        username = self.username_entry.get()
        password = self.password_entry.get()
        
        if username in self.users and self.users[username]["password"] == password:
            self.current_user = username
            self.user_role = self.users[username]["role"]
            self.login_window.destroy()
            self.load_data()
            self.create_main_interface()
            self.root.deiconify()  # Show main window
        else:
            messagebox.showerror("Lỗi", "Tên đăng nhập hoặc mật khẩu không đúng")
            self.password_entry.delete(0, tk.END)
            self.password_entry.focus_set()
    
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
        # Configure main window style
        self.root.configure(bg=self.bg_color)
        
        # Header
        header_frame = ttk.Frame(self.root, style='TFrame')
        header_frame.pack(fill=tk.X, padx=10, pady=10)
        
        ttk.Label(header_frame, text="COFFEE POS SYSTEM", 
                 font=('Arial', 18, 'bold'), 
                 foreground=self.primary_color).pack(side=tk.LEFT)
        
        user_info = ttk.Label(header_frame, 
                            text=f"Xin chào, {self.current_user} ({self.user_role})",
                            font=('Arial', 10),
                            foreground=self.secondary_color)
        user_info.pack(side=tk.RIGHT)
        
        # Tạo notebook (tab)
        self.notebook = ttk.Notebook(self.root)
        self.notebook.pack(fill=tk.BOTH, expand=True, padx=10, pady=(0, 10))
        
        # Tab POS
        self.pos_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.pos_tab, text="POS")
        self.create_pos_tab()
        
        # Tab Quản lý
        self.manage_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.manage_tab, text="Quản lý")
        self.create_manage_tab()
        
        # Tab Báo cáo doanh thu
        self.report_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.report_tab, text="Báo cáo doanh thu")
        self.create_report_tab()
        
        # Tab Khuyến mãi
        self.promotion_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.promotion_tab, text="Khuyến mãi")
        self.create_promotion_tab()
        
        # Tab Quản lý kho
        self.inventory_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.inventory_tab, text="Quản lý kho")
        self.create_inventory_tab()
        
        # Tab Cấu hình in
        self.print_tab = ttk.Frame(self.notebook, style='TFrame')
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
        # Main frame
        main_frame = ttk.Frame(self.pos_tab, style='TFrame')
        main_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Frame bàn
        table_frame = ttk.LabelFrame(main_frame, text=f"Danh sách bàn (1-{self.max_tables})")
        table_frame.pack(side=tk.LEFT, fill=tk.Y, padx=5, pady=5)
        
        # Scrollbar cho danh sách bàn
        table_canvas = tk.Canvas(table_frame, bg=self.bg_color, highlightthickness=0)
        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=table_canvas.yview)
        scrollable_frame = ttk.Frame(table_canvas, style='TFrame')
        
        scrollable_frame.bind(
            "<Configure>",
            lambda e: table_canvas.configure(
                scrollregion=table_canvas.bbox("all")
            )
        )
        
        table_canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        table_canvas.configure(yscrollcommand=scrollbar.set)
        
        table_canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        self.table_buttons = []
        for i in range(1, self.max_tables + 1):
            btn = tk.Button(scrollable_frame, text=f"Bàn {i}\n0 đ", width=10, height=3,
                          relief=tk.RAISED, bd=2, font=('Arial', 10),
                          command=lambda i=i: self.select_table(i))
            btn.pack(padx=5, pady=5)
            self.table_buttons.append(btn)
            self.update_table_button(i)
        
        # Main content frame
        content_frame = ttk.Frame(main_frame, style='TFrame')
        content_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Frame đồ uống
        drink_frame = ttk.LabelFrame(content_frame, text="Đồ uống")
        drink_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Nhóm đồ uống
        group_frame = ttk.Frame(drink_frame, style='TFrame')
        group_frame.pack(fill=tk.X, pady=5)
        
        groups = ["Tất cả"] + list(self.drinks.keys())
        for group in groups:
            btn = tk.Button(group_frame, text=group, width=10,
                          relief=tk.RAISED, bd=1, font=('Arial', 9),
                          command=lambda g=group: self.filter_drinks(g))
            btn.pack(side=tk.LEFT, padx=2)
        
        # Danh sách đồ uống
        self.drink_canvas = tk.Canvas(drink_frame, bg=self.bg_color, highlightthickness=0)
        self.drink_canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        
        scrollbar = ttk.Scrollbar(drink_frame, orient=tk.VERTICAL, command=self.drink_canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        self.drink_canvas.configure(yscrollcommand=scrollbar.set)
        self.drink_frame = ttk.Frame(self.drink_canvas, style='TFrame')
        self.drink_canvas.create_window((0, 0), window=self.drink_frame, anchor="nw")
        
        self.filter_drinks("Tất cả")
        
        # Right side frame
        right_frame = ttk.Frame(content_frame, style='TFrame')
        right_frame.pack(side=tk.RIGHT, fill=tk.Y, padx=5, pady=5)
        
        # Frame hóa đơn
        order_frame = ttk.LabelFrame(right_frame, text="Hóa đơn")
        order_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        self.order_listbox = tk.Listbox(order_frame, width=40, height=20,
                                      font=('Arial', 10), bd=2, relief=tk.SUNKEN)
        self.order_listbox.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        
        # Frame thanh toán
        payment_frame = ttk.LabelFrame(right_frame, text="Thanh toán")
        payment_frame.pack(fill=tk.X, padx=5, pady=5)
        
        # Phương thức thanh toán
        ttk.Label(payment_frame, text="Phương thức:").pack(anchor=tk.W, padx=5, pady=2)
        self.payment_method = ttk.Combobox(payment_frame, values=self.payment_methods, state="readonly")
        self.payment_method.pack(fill=tk.X, padx=5, pady=2)
        self.payment_method.current(0)
        
        # Khuyến mãi
        ttk.Label(payment_frame, text="Khuyến mãi:").pack(anchor=tk.W, padx=5, pady=2)
        self.promotion_combobox = ttk.Combobox(payment_frame, values=list(self.promotions.keys()), state="readonly")
        self.promotion_combobox.pack(fill=tk.X, padx=5, pady=2)
        
        # Tổng tiền
        self.total_label = tk.Label(payment_frame, text="Tổng: 0 đ", 
                                   font=("Arial", 12, "bold"), fg=self.primary_color)
        self.total_label.pack(pady=10)
        
        # Nút chức năng
        btn_frame = ttk.Frame(right_frame, style='TFrame')
        btn_frame.pack(fill=tk.X, pady=5)
        
        ttk.Button(btn_frame, text="Xóa món", style='Danger.TButton', 
                  command=self.delete_item).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        ttk.Button(btn_frame, text="Thanh toán", style='Success.TButton', 
                  command=self.checkout).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        ttk.Button(btn_frame, text="In hóa đơn", style='Primary.TButton', 
                  command=self.print_bill).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        ttk.Button(btn_frame, text="Chỉnh sửa bill", style='Warning.TButton', 
                  command=self.edit_bill).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    
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
            btn = tk.Button(self.drink_frame, text=f"{name}\n{price:,} đ", 
                          width=15, height=2, font=('Arial', 9),
                          relief=tk.RAISED, bd=2,
                          command=lambda n=name, p=price: self.add_to_order(n, p))
            btn.grid(row=idx//3, column=idx%3, padx=5, pady=5, sticky="nsew")
        
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
            btn.config(text=btn_text, bg=self.table_colors["selected"], fg="white")
        elif total > 0:
            btn.config(text=btn_text, bg=self.table_colors["occupied"], fg="black")
        else:
            btn.config(text=btn_text, bg=self.table_colors["default"], fg="black")
    
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
                self.table_orders[self.selected_table][i] = (name, qty+1, price)
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
        
        self.order_listbox.insert(tk.END, "-"*40)
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
        
        payment_method = self.payment_method.get()
        
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
        if not self.selected_table or not self.table_orders[self.selected_table]:
            messagebox.showwarning("Cảnh báo", "Không có hóa đơn để in")
            return
            
        filename = f"bill_table_{self.selected_table}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.pdf"
        c = canvas.Canvas(filename, pagesize=A4)
        width, height = A4
        
        # Header
        styles = getSampleStyleSheet()
        
        # Logo
        if self.bill_template["logo"] and os.path.exists(self.bill_template["logo"]):
            try:
                logo = Image(self.bill_template["logo"], width=100, height=50)
                logo.drawOn(c, 50, height-80)
            except:
                pass
        
        c.setFont("Helvetica-Bold", 16)
        c.drawCentredString(width/2, height-50, self.bill_template["header"])
        c.setFont("Helvetica", 10)
        c.drawCentredString(width/2, height-70, self.bill_template["address"])
        c.drawCentredString(width/2, height-85, self.bill_template["phone"])
        
        c.drawString(50, height-120, f"Bàn: {self.selected_table}")
        c.drawString(50, height-135, f"Ngày: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
        c.drawString(50, height-150, f"Nhân viên: {self.current_user}")
        
        # Chi tiết hóa đơn
        data = [["Tên món", "SL", "Đơn giá", "Thành tiền"]]
        
        total = 0
        for name, qty, price in self.table_orders[self.selected_table]:
            subtotal = qty * price
            total += subtotal
            data.append([name, str(qty), f"{price:,} đ", f"{subtotal:,} đ"])
        
        # Áp dụng khuyến mãi
        promotion = self.promotion_combobox.get()
        discount = 0
        if promotion and promotion in self.promotions:
            discount = self.promotions[promotion]["discount"]
            discount_amount = total * discount / 100
            total -= discount_amount
            data.append(["Khuyến mãi", f"{discount}%", f"-{discount_amount:,.0f} đ", ""])
        
        # Phương thức thanh toán
        payment_method = self.payment_method.get()
        data.append(["Phương thức thanh toán", payment_method, "", ""])
        
        # Tạo bảng
        table = Table(data, colWidths=[200, 50, 100, 100])
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 12),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
            ('GRID', (0, 0), (-1, -1), 1, colors.grey),
        ]))
        
        # Vẽ bảng
        table.wrapOn(c, width, height)
        table.drawOn(c, 50, height-300)
        
        # Tổng cộng
        c.setFont("Helvetica-Bold", 14)
        c.drawString(350, height-350, f"Tổng cộng: {total:,.0f} đ")
        
        # Chân trang
        c.setFont("Helvetica", 10)
        c.drawCentredString(width/2, 50, self.bill_template["footer"])
        
        c.save()
        
        # In ra máy in thật nếu được chọn
        if self.selected_printer and self.selected_printer in self.printers:
            try:
                if os.name == 'nt':  # Windows
                    win32api.ShellExecute(0, "print", filename, f'/d:"{self.selected_printer}"', ".", 0)
                else:
                    # Linux/Mac - cần cài đặt lpr
                    os.system(f'lpr -P {self.selected_printer} {filename}')
            except Exception as e:
                messagebox.showerror("Lỗi in", f"Không thể in: {str(e)}")
        
        messagebox.showinfo("Thành công", f"Đã lưu hóa đơn thành {filename}")
        
        # Mở file PDF
        if os.name == 'nt':  # Windows
            os.startfile(filename)
        else:  # Mac/Linux
            opener = "open" if sys.platform == "darwin" else "xdg-open"
            subprocess.call([opener, filename])
    
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
	    tk.Button(btn_frame, text="Hủy", command=edit_window.destroy).pack(side=tk.LEFT, padx=5)

def edit_quantity(self):
    selection = self.edit_listbox.curselection()
    if not selection:
        return
        
    index = selection[0]
    if index >= len(self.table_orders[self.selected_table]):
        return
        
    name, qty, price = self.table_orders[self.selected_table][index]
    new_qty = simpledialog.askinteger("Sửa số lượng", f"Nhập số lượng mới cho {name}:", 
                                    initialvalue=qty, minvalue=1)
    
    if new_qty and new_qty != qty:
        # Cập nhật tồn kho
        if name in self.inventory:
            diff = new_qty - qty
            self.inventory[name]["quantity"] -= diff
        
        # Cập nhật đơn hàng
        self.table_orders[self.selected_table][index] = (name, new_qty, price)
        self.table_totals[self.selected_table] += (new_qty - qty) * price
        
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
    
    self.edit_listbox.delete(index)

def save_edit(self, window):
    self.update_order_listbox()
    self.update_table_button(self.selected_table)
    self.save_data()
    window.destroy()
    messagebox.showinfo("Thành công", "Đã cập nhật hóa đơn")

def create_manage_tab(self):
    # Main frame
    main_frame = ttk.Frame(self.manage_tab, style='TFrame')
    main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
    
    # Notebook for management sections
    manage_notebook = ttk.Notebook(main_frame)
    manage_notebook.pack(fill=tk.BOTH, expand=True)
    
    # Drink management tab
    drink_manage_frame = ttk.Frame(manage_notebook, style='TFrame')
    manage_notebook.add(drink_manage_frame, text="Quản lý đồ uống")
    self.create_drink_manage_tab(drink_manage_frame)
    
    # Table management tab
    table_manage_frame = ttk.Frame(manage_notebook, style='TFrame')
    manage_notebook.add(table_manage_frame, text="Quản lý bàn")
    self.create_table_manage_tab(table_manage_frame)
    
    # User management tab
    user_manage_frame = ttk.Frame(manage_notebook, style='TFrame')
    manage_notebook.add(user_manage_frame, text="Quản lý người dùng")
    self.create_user_manage_tab(user_manage_frame)

def create_drink_manage_tab(self, parent):
    # Left frame - drink categories
    left_frame = ttk.Frame(parent, style='TFrame')
    left_frame.pack(side=tk.LEFT, fill=tk.Y, padx=5, pady=5)
    
    tk.Label(left_frame, text="Danh mục đồ uống", font=("Arial", 12, "bold")).pack(pady=5)
    
    self.category_listbox = tk.Listbox(left_frame, width=20, height=15)
    self.category_listbox.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
    
    for category in self.drinks.keys():
        self.category_listbox.insert(tk.END, category)
    
    btn_frame = ttk.Frame(left_frame, style='TFrame')
    btn_frame.pack(fill=tk.X, pady=5)
    
    ttk.Button(btn_frame, text="Thêm", style='Success.TButton', 
              command=self.add_category).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Sửa", style='Primary.TButton', 
              command=self.edit_category).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xóa", style='Danger.TButton', 
              command=self.delete_category).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    
    # Right frame - drinks in category
    right_frame = ttk.Frame(parent, style='TFrame')
    right_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=5, pady=5)
    
    tk.Label(right_frame, text="Danh sách đồ uống", font=("Arial", 12, "bold")).pack(pady=5)
    
    self.drink_listbox = tk.Listbox(right_frame, width=40, height=15)
    self.drink_listbox.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
    
    btn_frame = ttk.Frame(right_frame, style='TFrame')
    btn_frame.pack(fill=tk.X, pady=5)
    
    ttk.Button(btn_frame, text="Thêm", style='Success.TButton', 
              command=self.add_drink).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Sửa", style='Primary.TButton', 
              command=self.edit_drink).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xóa", style='Danger.TButton', 
              command=self.delete_drink).pack(side=tk.LEFT, padx=2, fill=tk.X, expand=True)
    
    # Bind category selection event
    self.category_listbox.bind('<<ListboxSelect>>', self.update_drink_list)

def add_category(self):
    category = simpledialog.askstring("Thêm danh mục", "Nhập tên danh mục mới:")
    if category and category not in self.drinks:
        self.drinks[category] = []
        self.category_listbox.insert(tk.END, category)
        self.save_data()
        self.filter_drinks("Tất cả")  # Refresh POS tab

def edit_category(self):
    selection = self.category_listbox.curselection()
    if not selection:
        return
        
    old_category = self.category_listbox.get(selection)
    new_category = simpledialog.askstring("Sửa danh mục", "Nhập tên mới:", 
                                         initialvalue=old_category)
    if new_category and new_category != old_category:
        self.drinks[new_category] = self.drinks.pop(old_category)
        self.category_listbox.delete(selection)
        self.category_listbox.insert(selection, new_category)
        self.save_data()
        self.filter_drinks("Tất cả")  # Refresh POS tab

def delete_category(self):
    selection = self.category_listbox.curselection()
    if not selection:
        return
        
    category = self.category_listbox.get(selection)
    if messagebox.askyesno("Xác nhận", f"Xóa danh mục '{category}' và tất cả đồ uống trong đó?"):
        del self.drinks[category]
        self.category_listbox.delete(selection)
        self.drink_listbox.delete(0, tk.END)
        self.save_data()
        self.filter_drinks("Tất cả")  # Refresh POS tab

def update_drink_list(self, event):
    selection = self.category_listbox.curselection()
    if not selection:
        return
        
    category = self.category_listbox.get(selection)
    self.drink_listbox.delete(0, tk.END)
    
    for name, price, _ in self.drinks[category]:
        self.drink_listbox.insert(tk.END, f"{name} - {price:,} đ")

def add_drink(self):
    selection = self.category_listbox.curselection()
    if not selection:
        messagebox.showwarning("Cảnh báo", "Vui lòng chọn danh mục trước")
        return
        
    category = self.category_listbox.get(selection)
    
    name = simpledialog.askstring("Thêm đồ uống", "Tên đồ uống:")
    if not name:
        return
        
    price = simpledialog.askinteger("Thêm đồ uống", "Giá tiền:", minvalue=1000)
    if not price:
        return
        
    self.drinks[category].append((name, price, None))
    self.update_drink_list(None)
    self.save_data()
    self.filter_drinks("Tất cả")  # Refresh POS tab

def edit_drink(self):
    cat_selection = self.category_listbox.curselection()
    drink_selection = self.drink_listbox.curselection()
    if not cat_selection or not drink_selection:
        return
        
    category = self.category_listbox.get(cat_selection)
    drink_index = drink_selection[0]
    old_name, old_price, _ = self.drinks[category][drink_index]
    
    name = simpledialog.askstring("Sửa đồ uống", "Tên mới:", initialvalue=old_name)
    if not name:
        return
        
    price = simpledialog.askinteger("Sửa đồ uống", "Giá mới:", 
                                  initialvalue=old_price, minvalue=1000)
    if not price:
        return
        
    self.drinks[category][drink_index] = (name, price, None)
    self.update_drink_list(None)
    self.save_data()
    self.filter_drinks("Tất cả")  # Refresh POS tab

def delete_drink(self):
    cat_selection = self.category_listbox.curselection()
    drink_selection = self.drink_listbox.curselection()
    if not cat_selection or not drink_selection:
        return
        
    category = self.category_listbox.get(cat_selection)
    drink_index = drink_selection[0]
    name, _, _ = self.drinks[category][drink_index]
    
    if messagebox.askyesno("Xác nhận", f"Xóa đồ uống '{name}'?"):
        del self.drinks[category][drink_index]
        self.update_drink_list(None)
        self.save_data()
        self.filter_drinks("Tất cả")  # Refresh POS tab

def create_table_manage_tab(self, parent):
    tk.Label(parent, text="Số lượng bàn hiện tại: {}".format(self.max_tables), 
           font=("Arial", 12)).pack(pady=10)
    
    tk.Label(parent, text="Nhập số lượng bàn mới:").pack()
    
    self.new_table_count = tk.Entry(parent)
    self.new_table_count.pack(pady=5)
    self.new_table_count.insert(0, str(self.max_tables))
    
    ttk.Button(parent, text="Cập nhật", style='Primary.TButton', 
              command=self.update_table_count).pack(pady=10)
    
    # Table status
    status_frame = ttk.LabelFrame(parent, text="Trạng thái các bàn")
    status_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
    
    self.table_status_text = tk.Text(status_frame, height=10, wrap=tk.WORD)
    self.table_status_text.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
    
    self.update_table_status()

def update_table_count(self):
    try:
        new_count = int(self.new_table_count.get())
        if new_count < 1:
            raise ValueError
            
        # Update data
        if new_count > self.max_tables:
            # Add new tables
            for i in range(self.max_tables + 1, new_count + 1):
                self.table_orders[i] = []
                self.table_totals[i] = 0
        else:
            # Remove tables (but keep their data)
            for i in range(new_count + 1, self.max_tables + 1):
                if i in self.table_orders:
                    del self.table_orders[i]
                if i in self.table_totals:
                    del self.table_totals[i]
        
        self.max_tables = new_count
        self.save_data()
        
        # Update POS tab
        for i in range(1, self.max_tables + 1):
            if i not in self.table_orders:
                self.table_orders[i] = []
            if i not in self.table_totals:
                self.table_totals[i] = 0
        
        # Recreate POS tab
        self.notebook.forget(self.pos_tab)
        self.pos_tab = ttk.Frame(self.notebook, style='TFrame')
        self.notebook.add(self.pos_tab, text="POS")
        self.create_pos_tab()
        
        messagebox.showinfo("Thành công", f"Đã cập nhật số lượng bàn thành {self.max_tables}")
        self.update_table_status()
    except ValueError:
        messagebox.showerror("Lỗi", "Vui lòng nhập số nguyên dương")

def update_table_status(self):
    self.table_status_text.delete(1.0, tk.END)
    
    occupied = 0
    for i in range(1, self.max_tables + 1):
        if self.table_totals[i] > 0:
            occupied += 1
            status = "Có khách"
        else:
            status = "Trống"
        
        self.table_status_text.insert(tk.END, f"Bàn {i}: {status} - Tổng: {self.table_totals[i]:,} đ\n")
    
    self.table_status_text.insert(tk.END, f"\nTổng số bàn: {self.max_tables}\n")
    self.table_status_text.insert(tk.END, f"Bàn có khách: {occupied}\n")
    self.table_status_text.insert(tk.END, f"Bàn trống: {self.max_tables - occupied}")

def create_user_manage_tab(self, parent):
    # User list
    tk.Label(parent, text="Danh sách người dùng", font=("Arial", 12, "bold")).pack(pady=5)
    
    self.user_listbox = tk.Listbox(parent, width=40, height=10)
    self.user_listbox.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
    
    self.update_user_list()
    
    # Buttons
    btn_frame = ttk.Frame(parent, style='TFrame')
    btn_frame.pack(fill=tk.X, pady=10)
    
    ttk.Button(btn_frame, text="Thêm", style='Success.TButton', 
              command=self.add_user).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Sửa", style='Primary.TButton', 
              command=self.edit_user).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xóa", style='Danger.TButton', 
              command=self.delete_user).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Đổi mật khẩu", style='Warning.TButton', 
              command=self.change_password).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)

def update_user_list(self):
    self.user_listbox.delete(0, tk.END)
    for username, data in self.users.items():
        self.user_listbox.insert(tk.END, f"{username} ({data['role']})")

def add_user(self):
    username = simpledialog.askstring("Thêm người dùng", "Tên đăng nhập:")
    if not username:
        return
        
    if username in self.users:
        messagebox.showerror("Lỗi", "Tên đăng nhập đã tồn tại")
        return
        
    password = simpledialog.askstring("Thêm người dùng", "Mật khẩu:", show="*")
    if not password:
        return
        
    role = simpledialog.askstring("Thêm người dùng", "Vai trò (admin/staff):")
    if role not in ["admin", "staff"]:
        messagebox.showerror("Lỗi", "Vai trò phải là admin hoặc staff")
        return
        
    self.users[username] = {"password": password, "role": role}
    self.save_data()
    self.update_user_list()
    messagebox.showinfo("Thành công", "Đã thêm người dùng mới")

def edit_user(self):
    selection = self.user_listbox.curselection()
    if not selection:
        return
        
    username = self.user_listbox.get(selection).split()[0]
    
    new_role = simpledialog.askstring("Sửa người dùng", "Vai trò mới (admin/staff):", 
                                    initialvalue=self.users[username]["role"])
    if new_role and new_role in ["admin", "staff"]:
        self.users[username]["role"] = new_role
        self.save_data()
        self.update_user_list()
        messagebox.showinfo("Thành công", "Đã cập nhật vai trò")

def delete_user(self):
    selection = self.user_listbox.curselection()
    if not selection:
        return
        
    username = self.user_listbox.get(selection).split()[0]
    
    if username == self.current_user:
        messagebox.showerror("Lỗi", "Không thể xóa tài khoản đang đăng nhập")
        return
        
    if messagebox.askyesno("Xác nhận", f"Xóa người dùng '{username}'?"):
        del self.users[username]
        self.save_data()
        self.update_user_list()

def change_password(self):
    selection = self.user_listbox.curselection()
    if not selection:
        return
        
    username = self.user_listbox.get(selection).split()[0]
    
    new_password = simpledialog.askstring("Đổi mật khẩu", "Mật khẩu mới:", show="*")
    if new_password:
        self.users[username]["password"] = new_password
        self.save_data()
        messagebox.showinfo("Thành công", "Đã đổi mật khẩu")

def create_report_tab(self):
    # Date selection
    date_frame = ttk.Frame(self.report_tab, style='TFrame')
    date_frame.pack(fill=tk.X, padx=10, pady=10)
    
    ttk.Label(date_frame, text="Chọn ngày:").pack(side=tk.LEFT)
    
    self.report_date = ttk.Entry(date_frame)
    self.report_date.pack(side=tk.LEFT, padx=5)
    self.report_date.insert(0, datetime.now().strftime("%d/%m/%Y"))
    
    ttk.Button(date_frame, text="Xem báo cáo", style='Primary.TButton', 
              command=self.generate_report).pack(side=tk.LEFT)
    
    # Report display
    report_frame = ttk.Frame(self.report_tab, style='TFrame')
    report_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
    
    self.report_text = tk.Text(report_frame, wrap=tk.WORD, font=("Arial", 10))
    self.report_text.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
    
    # Export buttons
    btn_frame = ttk.Frame(self.report_tab, style='TFrame')
    btn_frame.pack(fill=tk.X, padx=10, pady=5)
    
    ttk.Button(btn_frame, text="Xuất PDF", style='Primary.TButton', 
              command=self.export_report_pdf).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xuất Excel", style='Success.TButton', 
              command=self.export_report_excel).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)

def generate_report(self):
    date = self.report_date.get()
    
    try:
        datetime.strptime(date, "%d/%m/%Y")
    except ValueError:
        messagebox.showerror("Lỗi", "Định dạng ngày không hợp lệ. Vui lòng nhập dd/mm/yyyy")
        return
        
    self.report_text.delete(1.0, tk.END)
    
    if date in self.daily_reports:
        report = self.daily_reports[date]
        total = report["total"]
        orders = report["orders"]
        
        self.report_text.insert(tk.END, f"BÁO CÁO DOANH THU NGÀY {date}\n\n")
        self.report_text.insert(tk.END, f"Tổng số đơn: {orders}\n")
        self.report_text.insert(tk.END, f"Tổng doanh thu: {total:,.0f} đ\n\n")
        
        self.report_text.insert(tk.END, "Phương thức thanh toán:\n")
        for method, amount in report["payment_methods"].items():
            if amount > 0:
                self.report_text.insert(tk.END, f"- {method}: {amount:,.0f} đ\n")
    else:
        self.report_text.insert(tk.END, f"Không có dữ liệu cho ngày {date}")

def export_report_pdf(self):
    filename = filedialog.asksaveasfilename(defaultextension=".pdf", 
                                          filetypes=[("PDF files", "*.pdf")])
    if not filename:
        return
        
    c = canvas.Canvas(filename, pagesize=A4)
    width, height = A4
    
    # Header
    c.setFont("Helvetica-Bold", 16)
    c.drawCentredString(width/2, height-50, "BÁO CÁO DOANH THU")
    c.setFont("Helvetica", 12)
    c.drawCentredString(width/2, height-80, f"Ngày: {self.report_date.get()}")
    
    # Report content
    report_text = self.report_text.get(1.0, tk.END)
    text = c.beginText(50, height-120)
    text.setFont("Helvetica", 10)
    
    for line in report_text.split('\n'):
        text.textLine(line)
    
    c.drawText(text)
    c.save()
    
    messagebox.showinfo("Thành công", f"Đã xuất báo cáo thành {filename}")

def export_report_excel(self):
    messagebox.showinfo("Thông báo", "Chức năng xuất Excel sẽ được phát triển sau")

def create_promotion_tab(self):
    # Promotion list
    tk.Label(self.promotion_tab, text="Danh sách khuyến mãi", 
            font=("Arial", 12, "bold")).pack(pady=5)
    
    self.promotion_listbox = tk.Listbox(self.promotion_tab, width=60, height=10)
    self.promotion_listbox.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
    
    self.update_promotion_list()
    
    # Buttons
    btn_frame = ttk.Frame(self.promotion_tab, style='TFrame')
    btn_frame.pack(fill=tk.X, pady=10)
    
    ttk.Button(btn_frame, text="Thêm", style='Success.TButton', 
              command=self.add_promotion).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Sửa", style='Primary.TButton', 
              command=self.edit_promotion).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xóa", style='Danger.TButton', 
              command=self.delete_promotion).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)

def update_promotion_list(self):
    self.promotion_listbox.delete(0, tk.END)
    for name, details in self.promotions.items():
        start = details["start_date"]
        end = details["end_date"]
        discount = details["discount"]
        self.promotion_listbox.insert(tk.END, f"{name}: Giảm {discount}% ({start} đến {end})")

def add_promotion(self):
    name = simpledialog.askstring("Thêm khuyến mãi", "Tên chương trình:")
    if not name:
        return
        
    discount = simpledialog.askinteger("Thêm khuyến mãi", "Phần trăm giảm giá:", 
                                     minvalue=1, maxvalue=100)
    if not discount:
        return
        
    start_date = simpledialog.askstring("Thêm khuyến mãi", "Ngày bắt đầu (dd/mm/yyyy):")
    if not start_date:
        return
        
    end_date = simpledialog.askstring("Thêm khuyến mãi", "Ngày kết thúc (dd/mm/yyyy):")
    if not end_date:
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
    
    # Update POS tab combobox
    self.promotion_combobox['values'] = list(self.promotions.keys())

def edit_promotion(self):
    selection = self.promotion_listbox.curselection()
    if not selection:
        return
        
    old_name = self.promotion_listbox.get(selection).split(":")[0]
    details = self.promotions[old_name]
    
    name = simpledialog.askstring("Sửa khuyến mãi", "Tên mới:", 
                                initialvalue=old_name)
    if not name:
        return
        
    discount = simpledialog.askinteger("Sửa khuyến mãi", "Phần trăm giảm giá mới:", 
                                     initialvalue=details["discount"], 
                                     minvalue=1, maxvalue=100)
    if not discount:
        return
        
    start_date = simpledialog.askstring("Sửa khuyến mãi", "Ngày bắt đầu mới:", 
                                      initialvalue=details["start_date"])
    if not start_date:
        return
        
    end_date = simpledialog.askstring("Sửa khuyến mãi", "Ngày kết thúc mới:", 
                                    initialvalue=details["end_date"])
    if not end_date:
        return
        
    try:
        datetime.strptime(start_date, "%d/%m/%Y")
        datetime.strptime(end_date, "%d/%m/%Y")
    except ValueError:
        messagebox.showerror("Lỗi", "Định dạng ngày không hợp lệ. Vui lòng nhập dd/mm/yyyy")
        return
        
    if old_name != name:
        del self.promotions[old_name]
        
    self.promotions[name] = {
        "discount": discount,
        "start_date": start_date,
        "end_date": end_date
    }
    
    self.save_data()
    self.update_promotion_list()
    
    # Update POS tab combobox
    self.promotion_combobox['values'] = list(self.promotions.keys())

def delete_promotion(self):
    selection = self.promotion_listbox.curselection()
    if not selection:
        return
        
    name = self.promotion_listbox.get(selection).split(":")[0]
    
    if messagebox.askyesno("Xác nhận", f"Xóa khuyến mãi '{name}'?"):
        del self.promotions[name]
        self.save_data()
        self.update_promotion_list()
        
        # Update POS tab combobox
        self.promotion_combobox['values'] = list(self.promotions.keys())

def create_inventory_tab(self):
    # Inventory list
    tk.Label(self.inventory_tab, text="Quản lý kho hàng", 
            font=("Arial", 12, "bold")).pack(pady=5)
    
    self.inventory_tree = ttk.Treeview(self.inventory_tab, columns=("name", "quantity", "unit"), 
                                     show="headings")
    self.inventory_tree.heading("name", text="Tên hàng")
    self.inventory_tree.heading("quantity", text="Số lượng")
    self.inventory_tree.heading("unit", text="Đơn vị")
    self.inventory_tree.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
    
    self.update_inventory_list()
    
    # Buttons
    btn_frame = ttk.Frame(self.inventory_tab, style='TFrame')
    btn_frame.pack(fill=tk.X, pady=10)
    
    ttk.Button(btn_frame, text="Thêm", style='Success.TButton', 
              command=self.add_inventory_item).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Sửa", style='Primary.TButton', 
              command=self.edit_inventory_item).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Xóa", style='Danger.TButton', 
              command=self.delete_inventory_item).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
    ttk.Button(btn_frame, text="Nhập hàng", style='Warning.TButton', 
              command=self.import_inventory).pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)

def update_inventory_list(self):
    for item in self.inventory_tree.get_children():
        self.inventory_tree.delete(item)
        
    for name, details in self.inventory.items():
        self.inventory_tree.insert("", tk.END, values=(name, details["quantity"], details["unit"]))

def add_inventory_item(self):
    name = simpledialog.askstring("Thêm hàng vào kho", "Tên hàng:")
    if not name:
        return
        
    quantity = simpledialog.askinteger("Thêm hàng vào kho", "Số lượng:", minvalue=0)
    if quantity is None:
        return
        
    unit = simpledialog.askstring("Thêm hàng vào kho", "Đơn vị tính:")
    if not unit:
        return
        
    self.inventory[name] = {"quantity": quantity, "unit": unit}
    self.save_data()
    self.update_inventory_list()

def edit_inventory_item(self):
    selection = self.inventory_tree.selection()
    if not selection:
        return
        
    item = self.inventory_tree.item(selection)
    old_name = item['values'][0]
    old_quantity = item['values'][1]
    old_unit = item['values'][2]
    
    name = simpledialog.askstring("Sửa hàng trong kho", "Tên mới:", 
                                initialvalue=old_name)
    if not name:
        return
        
    quantity = simpledialog.askinteger("Sửa hàng trong kho", "Số lượng mới:", 
                                     initialvalue=old_quantity, minvalue=0)
    if quantity is None:
        return
        
    unit = simpledialog.askstring("Sửa hàng trong kho", "Đơn vị mới:", 
                                initialvalue=old_unit)
    if not unit:
        return
        
    if old_name != name:
        del self.inventory[old_name]
        
    self.inventory[name] = {"quantity": quantity, "unit": unit}
    self.save_data()
    self.update_inventory_list()

def delete_inventory_item(self):
    selection = self.inventory_tree.selection()
    if not selection:
        return
        
    item = self.inventory_tree.item(selection)
    name = item['values'][0]
    
    if messagebox.askyesno("Xác nhận", f"Xóa '{name}' khỏi kho hàng?"):
        del self.inventory[name]
        self.save_data()
        self.update_inventory_list()

def import_inventory(self):
    selection = self.inventory_tree.selection()
    if not selection:
        return
        
    item = self.inventory_tree.item(selection)
    name = item['values'][0]
    current_qty = item['values'][1]
    
    add_qty = simpledialog.askinteger("Nhập hàng", f"Nhập số lượng {name} cần thêm:", 
                                    minvalue=1)
    if not add_qty:
        return
        
    self.inventory[name]["quantity"] += add_qty
    self.save_data()
    self.update_inventory_list()
    messagebox.showinfo("Thành công", f"Đã thêm {add_qty} {self.inventory[name]['unit']} {name} vào kho")

def create_print_tab(self):
    # Printer selection
    printer_frame = ttk.LabelFrame(self.print_tab, text="Máy in")
    printer_frame.pack(fill=tk.X, padx=10, pady=10)
    
    ttk.Label(printer_frame, text="Chọn máy in:").pack(anchor=tk.W, padx=5, pady=2)
    
    self.printer_combobox = ttk.Combobox(printer_frame, values=self.printers, state="readonly")
    self.printer_combobox.pack(fill=tk.X, padx=5, pady=2)
    
    if self.selected_printer and self.selected_printer in self.printers:
        self.printer_combobox.set(self.selected_printer)
    
    ttk.Button(printer_frame, text="Làm mới danh sách", 
              command=self.refresh_printers).pack(fill=tk.X, padx=5, pady=5)
    
    # Bill template
    template_frame = ttk.LabelFrame(self.print_tab, text="Mẫu hóa đơn")
    template_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
    
    # Header
    ttk.Label(template_frame, text="Tiêu đề:").grid(row=0, column=0, sticky=tk.W, padx=5, pady=2)
    self.bill_header = ttk.Entry(template_frame)
    self.bill_header.grid(row=0, column=1, sticky=tk.EW, padx=5, pady=2)
    self.bill_header.insert(0, self.bill_template["header"])
    
    # Address
    ttk.Label(template_frame, text="Địa chỉ:").grid(row=1, column=0, sticky=tk.W, padx=5, pady=2)
    self.bill_address = ttk.Entry(template_frame)
    self.bill_address
    self.bill_address.grid(row=1, column=1, sticky=tk.EW, padx=5, pady=2)
    self.bill_address.insert(0, self.bill_template["address"])
    
    # Phone
    ttk.Label(template_frame, text="Điện thoại:").grid(row=2, column=0, sticky=tk.W, padx=5, pady=2)
    self.bill_phone = ttk.Entry(template_frame)
    self.bill_phone.grid(row=2, column=1, sticky=tk.EW, padx=5, pady=2)
    self.bill_phone.insert(0, self.bill_template["phone"])
    
    # Footer
    ttk.Label(template_frame, text="Chân trang:").grid(row=3, column=0, sticky=tk.W, padx=5, pady=2)
    self.bill_footer = ttk.Entry(template_frame)
    self.bill_footer.grid(row=3, column=1, sticky=tk.EW, padx=5, pady=2)
    self.bill_footer.insert(0, self.bill_template["footer"])
    
    # Logo
    ttk.Label(template_frame, text="Logo:").grid(row=4, column=0, sticky=tk.W, padx=5, pady=2)
    self.logo_path = tk.StringVar(value=self.bill_template.get("logo", ""))
    ttk.Entry(template_frame, textvariable=self.logo_path, state='readonly').grid(row=4, column=1, sticky=tk.EW, padx=5, pady=2)
    ttk.Button(template_frame, text="Chọn ảnh", command=self.select_logo).grid(row=4, column=2, padx=5, pady=2)
    
    # Save button
    ttk.Button(self.print_tab, text="Lưu cấu hình", style='Primary.TButton', 
              command=self.save_print_config).pack(pady=10)
    
    # Configure grid weights
    template_frame.columnconfigure(1, weight=1)

def refresh_printers(self):
    self.get_system_printers()
    self.printer_combobox['values'] = self.printers
    if self.selected_printer in self.printers:
        self.printer_combobox.set(self.selected_printer)
    messagebox.showinfo("Thành công", "Đã làm mới danh sách máy in")

def select_logo(self):
    filename = filedialog.askopenfilename(title="Chọn ảnh logo", 
                                        filetypes=[("Image files", "*.png;*.jpg;*.jpeg")])
    if filename:
        self.logo_path.set(filename)

def save_print_config(self):
    self.selected_printer = self.printer_combobox.get()
    
    self.bill_template = {
        "header": self.bill_header.get(),
        "address": self.bill_address.get(),
        "phone": self.bill_phone.get(),
        "footer": self.bill_footer.get(),
        "logo": self.logo_path.get() if self.logo_path.get() else None
    }
    
    self.save_data()
    messagebox.showinfo("Thành công", "Đã lưu cấu hình in")

def run(self):
    self.root.mainloop()

if __name__ == "__main__":
    root = tk.Tk()
    app = CoffeePOSApp(root)
    app.run()
