import tkinter as tk
from tkinter import ttk, messagebox
import pyodbc


def get_connection():
    return pyodbc.connect(
        "DRIVER={ODBC Driver 17 for SQL Server};"
        "SERVER=localhost;DATABASE=Exam;Trusted_Connection=yes;"
    )


class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Обувной магазин")

        try:
            self.iconphoto(True, tk.PhotoImage(file="Icon.png"))
        except Exception:
            pass

        self.user = "Гость"
        self.role = "Гость"
        self.show_login()

    def clear(self):
        for w in self.winfo_children():
            w.destroy()

    def show_login(self):
        self.clear()
        self.state("normal")
        self.title("Авторизация")
        self.geometry("350x230")

        tk.Label(self, text="Обувной магазин").pack(pady=10)
        tk.Label(self, text="Логин").pack()
        self.login_entry = tk.Entry(self)
        self.login_entry.pack()

        tk.Label(self, text="Пароль").pack()
        self.pass_entry = tk.Entry(self, show="*")
        self.pass_entry.pack()

        tk.Button(self, text="Войти", width=20, command=self.do_login).pack(pady=8)
        tk.Button(self, text="Войти как гость", width=20, command=self.do_guest).pack()

    def do_login(self):
        login = self.login_entry.get().strip()
        password = self.pass_entry.get().strip()

        if not login or not password:
            messagebox.showerror("Ошибка", "Введите логин и пароль")
            return

        try:
            conn = get_connection()
            cur = conn.cursor()
            cur.execute("""
                SELECT u.Surname + ' ' + u.Name, r.Name
                FROM dbo.[User] u
                JOIN dbo.Role r ON u.RoleId = r.Id
                WHERE u.Login = ? AND u.Password = ?
            """, (login, password))
            row = cur.fetchone()
            conn.close()

            if row:
                self.user, self.role = row
                self.show_products()
            else:
                messagebox.showerror("Ошибка", "Неверный логин или пароль")

        except Exception as e:
            messagebox.showerror("Ошибка БД", str(e))

    def do_guest(self):
        self.user = "Гость"
        self.role = "Гость"
        self.show_products()

    def show_products(self):
        self.clear()
        self.title("Список товаров")
        self.state("zoomed")

        top = tk.Frame(self)
        top.pack(fill="x", padx=5, pady=5)

        tk.Label(top, text=f"{self.user} ({self.role})").pack(side="left")
        tk.Button(top, text="Выйти", command=self.show_login).pack(side="right", padx=3)

        if self.role in ("Менеджер", "Администратор"):
            tk.Button(top, text="Заказы", command=self.show_orders).pack(side="right", padx=3)

        if self.role == "Администратор":
            tk.Button(top, text="Удалить товар", command=self.delete_product).pack(side="right", padx=3)

        if self.role in ("Менеджер", "Администратор"):
            filt = tk.Frame(self)
            filt.pack(fill="x", padx=5, pady=3)

            tk.Label(filt, text="Поиск:").pack(side="left")
            self.search_entry = tk.Entry(filt, width=30)
            self.search_entry.pack(side="left", padx=5)
            self.search_entry.bind("<KeyRelease>", lambda e: self.load_products())

            tk.Button(filt, text="Сорт. по цене", command=self.toggle_sort).pack(side="left", padx=5)

        cols = ("ID", "Артикул", "Название", "Категория",
                "Производитель", "Поставщик", "Цена", "Остаток", "Скидка")
        self.products_tree = ttk.Treeview(self, columns=cols, show="headings")

        for c in cols:
            self.products_tree.heading(c, text=c)
            self.products_tree.column(c, width=120)

        self.products_tree.column("ID", width=70)
        self.products_tree.column("Артикул", width=120)

        self.products_tree.tag_configure("discount", background="#2E8B57", foreground="white")
        self.products_tree.tag_configure("empty", background="lightblue")
        self.products_tree.pack(fill="both", expand=True, padx=5, pady=5)

        self.sort_dir = "ASC"
        self.load_products()

    def toggle_sort(self):
        self.sort_dir = "DESC" if self.sort_dir == "ASC" else "ASC"
        self.load_products()

    def load_products(self):
        for i in self.products_tree.get_children():
            self.products_tree.delete(i)

        query = """
            SELECT
                p.Id,
                p.Article,
                p.Name,
                c.Name,
                pr.Name,
                pv.Name,
                p.Price,
                p.AmountInStock,
                p.Discount
            FROM dbo.Product p
            JOIN dbo.Category c ON p.CategoryId = c.Id
            JOIN dbo.Producer pr ON p.ProducerId = pr.Id
            JOIN dbo.Provider pv ON p.ProviderId = pv.Id
        """
        params = []

        if self.role in ("Менеджер", "Администратор"):
            text = self.search_entry.get().strip()
            if text:
                m = "%" + text + "%"
                query += """
                    WHERE p.Article LIKE ?
                       OR p.Name LIKE ?
                       OR c.Name LIKE ?
                       OR pr.Name LIKE ?
                """
                params = [m, m, m, m]
            query += f" ORDER BY p.Price {self.sort_dir}"
        else:
            query += " ORDER BY p.Name"

        try:
            conn = get_connection()
            cur = conn.cursor()
            cur.execute(query, params)
            rows = cur.fetchall()
            conn.close()

            for r in rows:
                pid, article, name, cat, prod, prov, price, stock, disc = r
                disc = disc or 0
                price = round(float(price), 2)

                if disc > 0:
                    new_price = round(price * (1 - disc / 100), 2)
                    price_str = f"{price} -> {new_price}"
                else:
                    price_str = str(price)

                tag = ""
                if stock == 0:
                    tag = "empty"
                elif disc > 15:
                    tag = "discount"

                self.products_tree.insert(
                    "",
                    "end",
                    values=(pid, article, name, cat, prod, prov, price_str, stock, f"{disc}%"),
                    tags=(tag,)
                )

        except Exception as e:
            messagebox.showerror("Ошибка", str(e))

    def delete_product(self):
        item = self.products_tree.focus()

        if not item:
            messagebox.showwarning("Внимание", "Выберите товар")
            return

        vals = self.products_tree.item(item)["values"]
        pid = vals[0]
        article = vals[1]
        name = vals[2]

        if not messagebox.askyesno("Подтверждение", f"Удалить товар '{name}' (артикул: {article})?"):
            return

        try:
            conn = get_connection()
            cur = conn.cursor()

            cur.execute("SELECT COUNT(*) FROM dbo.ProductInOrder WHERE ProductId = ?", (pid,))
            if cur.fetchone()[0] > 0:
                conn.close()
                messagebox.showerror("Ошибка", "Нельзя удалить товар, который есть в заказе")
                return

            cur.execute("DELETE FROM dbo.Product WHERE Id = ?", (pid,))
            conn.commit()
            conn.close()
            self.load_products()

        except Exception as e:
            messagebox.showerror("Ошибка", str(e))

    def show_orders(self):
        win = tk.Toplevel(self)
        win.title("Список заказов")
        win.geometry("850x500")
        win.grab_set()

        cols = ("ID", "Клиент", "Дата создания", "Дата доставки", "Статус", "Сумма")
        self.orders_tree = ttk.Treeview(win, columns=cols, show="headings")

        for c in cols:
            self.orders_tree.heading(c, text=c)
            self.orders_tree.column(c, width=130)

        self.orders_tree.pack(fill="both", expand=True, padx=5, pady=5)

        tk.Button(win, text="Редактировать", width=20, command=self.edit_order).pack(pady=5)

        self.load_orders()

    def load_orders(self):
        for i in self.orders_tree.get_children():
            self.orders_tree.delete(i)

        try:
            conn = get_connection()
            cur = conn.cursor()
            cur.execute("""
                SELECT
                    o.Id,
                    u.Surname + ' ' + u.Name,
                    o.CreationDate,
                    o.DeliveryDate,
                    s.Name,
                    (
                        SELECT SUM(pio.Amount * p.Price)
                        FROM dbo.ProductInOrder pio
                        JOIN dbo.Product p ON pio.ProductId = p.Id
                        WHERE pio.OrderId = o.Id
                    )
                FROM dbo.[Order] o
                JOIN dbo.[User] u ON o.UserId = u.Id
                JOIN dbo.Status s ON o.StatusId = s.Id
                ORDER BY o.Id DESC
            """)
            rows = cur.fetchall()
            conn.close()

            for r in rows:
                total = f"{round(float(r[5]), 2)} руб." if r[5] else "0 руб."
                self.orders_tree.insert("", "end", values=(r[0], r[1], r[2], r[3], r[4], total))

        except Exception as e:
            messagebox.showerror("Ошибка", str(e))

    def edit_order(self):
        item = self.orders_tree.focus()

        if not item:
            messagebox.showwarning("Внимание", "Выберите заказ")
            return

        vals = self.orders_tree.item(item)["values"]
        order_id = vals[0]

        try:
            conn = get_connection()
            cur = conn.cursor()
            cur.execute("SELECT StatusId FROM dbo.[Order] WHERE Id = ?", (order_id,))
            row = cur.fetchone()
            conn.close()
            cur_status = row[0] if row else 2
        except Exception:
            cur_status = 2

        win = tk.Toplevel(self)
        win.title(f"Редактирование заказа №{order_id}")
        win.geometry("320x150")
        win.grab_set()

        tk.Label(win, text="Статус (1=Завершён, 2=Новый):").pack(pady=10)
        status_entry = tk.Entry(win)
        status_entry.insert(0, str(cur_status))
        status_entry.pack()

        def save():
            try:
                s = int(status_entry.get().strip())
                conn = get_connection()
                cur = conn.cursor()
                cur.execute(
                    "UPDATE dbo.[Order] SET StatusId = ? WHERE Id = ?",
                    (s, order_id)
                )
                conn.commit()
                conn.close()
                win.destroy()
                self.load_orders()
            except Exception as e:
                messagebox.showerror("Ошибка", str(e))

        tk.Button(win, text="Сохранить", width=20, command=save).pack(pady=15)


if __name__ == "__main__":
    App().mainloop()



try:
    conn = pyodbc.connect(
        "DRIVER={ODBC Driver 17 for SQL Server};"
        "SERVER=localhost;"
        "DATABASE=Exam;"
        "Trusted_Connection=yes;"
    )
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    print("Подключение успешно")
    print(cursor.fetchone())
    conn.close()
except Exception as e:
    print("Ошибка подключения:", e)



drivers = pyodbc.drivers()

print("Установленные ODBC драйверы:")
for d in drivers:
    print(" -", d)

target = "ODBC Driver 17 for SQL Server"

if target in drivers:
    print(f"
{target} установлен")
else:
    print(f"
{target} НЕ установлен")

# Покажем подходящие драйверы для SQL Server
sql_drivers = [d for d in drivers if "SQL Server" in d.lower() or "SQL Server" in d]
print("
Драйверы, связанные с SQL Server:")
for d in sql_drivers:
    print(" -", d)



import pyodbc

def get_connection():
    return pyodbc.connect(
        "DRIVER={ODBC Driver 17 for SQL Server};"
        "SERVER=ИМЯ_СЕРВЕРА\\SQLEXPRESS;DATABASE=Exam;Trusted_Connection=yes;"
    )

# Тест подключения
try:
    conn = get_connection()
    print("✓ Подключение успешно!")
    
    # Проверка, что база доступна (даже пустая)
    cur = conn.cursor()
    cur.execute("SELECT 1 AS Test")
    result = cur.fetchone()
    print(f"✓ Запрос выполнен: {result[0]}")
    
    # Показать таблицы в базе (может быть пустой список)
    cur.execute("""
        SELECT TABLE_NAME 
        FROM INFORMATION_SCHEMA.TABLES 
        WHERE TABLE_TYPE = 'BASE TABLE'
    """)
    tables = cur.fetchall()
    if tables:
        print(f"✓ Таблиц в базе: {len(tables)}")
        for t in tables:
            print(f"  - {t[0]}")
    else:
        print("✓ База данных пуста (нет таблиц)")
    
    conn.close()
    print("✓ Подключение закрыто")
    
except Exception as e:
    print(f"✗ Ошибка подключения: {e}")