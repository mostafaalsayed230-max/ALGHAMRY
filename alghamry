import sqlite3
from datetime import datetime

# === إنشاء قاعدة البيانات ===
def init_db():
    conn = sqlite3.connect('factory.db')
    c = conn.cursor()

    # المواد الخام
    c.execute('''CREATE TABLE IF NOT EXISTS raw_materials (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL UNIQUE,
        unit TEXT NOT NULL,
        current_stock REAL NOT NULL DEFAULT 0,
        cost_per_unit REAL NOT NULL
    )''')

    # المنتجات
    c.execute('''CREATE TABLE IF NOT EXISTS products (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL UNIQUE,
        selling_price REAL NOT NULL
    )''')

    # وصفة الإنتاج (كم خامة لكل منتج)
    c.execute('''CREATE TABLE IF NOT EXISTS production_recipes (
        product_id INTEGER,
        raw_material_id INTEGER,
        quantity_needed REAL NOT NULL,
        FOREIGN KEY(product_id) REFERENCES products(id),
        FOREIGN KEY(raw_material_id) REFERENCES raw_materials(id),
        PRIMARY KEY(product_id, raw_material_id)
    )''')

    # سجل الإنتاج
    c.execute('''CREATE TABLE IF NOT EXISTS production_logs (
        id INTEGER PRIMARY KEY,
        product_id INTEGER,
        quantity_produced INTEGER NOT NULL,
        date TEXT NOT NULL,
        FOREIGN KEY(product_id) REFERENCES products(id)
    )''')

    # الفواتير
    c.execute('''CREATE TABLE IF NOT EXISTS invoices (
        id INTEGER PRIMARY KEY,
        customer_name TEXT NOT NULL,
        total_amount REAL NOT NULL,
        paid_amount REAL NOT NULL DEFAULT 0,
        date TEXT NOT NULL
    )''')

    # تفاصيل الفاتورة
    c.execute('''CREATE TABLE IF NOT EXISTS invoice_items (
        invoice_id INTEGER,
        product_id INTEGER,
        quantity INTEGER NOT NULL,
        price_per_unit REAL NOT NULL,
        FOREIGN KEY(invoice_id) REFERENCES invoices(id),
        FOREIGN KEY(product_id) REFERENCES products(id)
    )''')

    # المصروفات (مثل شراء خامات جديدة، كهرباء...)
    c.execute('''CREATE TABLE IF NOT EXISTS expenses (
        id INTEGER PRIMARY KEY,
        description TEXT NOT NULL,
        amount REAL NOT NULL,
        date TEXT NOT NULL
    )''')

    conn.commit()
    conn.close()

# === دوال مساعدة ===
def get_connection():
    return sqlite3.connect('factory.db')

# === 1. إضافة مادة خام ===
def add_raw_material():
    name = input("اسم المادة الخام: ").strip()
    unit = input("الوحدة (متر/كجم/قطعة...): ").strip()
    stock = float(input("الكمية الأولية: "))
    cost = float(input("تكلفة الوحدة: "))

    conn = get_connection()
    c = conn.cursor()
    try:
        c.execute("INSERT INTO raw_materials (name, unit, current_stock, cost_per_unit) VALUES (?, ?, ?, ?)",
                  (name, unit, stock, cost))
        conn.commit()
        print("✅ تمت إضافة المادة الخام بنجاح!")
    except sqlite3.IntegrityError:
        print("❌ هذه المادة موجودة مسبقًا!")
    conn.close()

# === 2. إضافة منتج ===
def add_product():
    name = input("اسم المنتج (مثلاً: كوتشي رياضي): ").strip()
    price = float(input("سعر البيع للوحدة: "))

    conn = get_connection()
    c = conn.cursor()
    try:
        c.execute("INSERT INTO products (name, selling_price) VALUES (?, ?)", (name, price))
        product_id = c.lastrowid

        # الآن أدخل وصفة الإنتاج
        print("أدخل وصفة الإنتاج (أدخل 0 لإنهاء):")
        while True:
            mat_name = input("اسم المادة الخام (أو 0 للخروج): ").strip()
            if mat_name == "0":
                break
            qty = float(input(f"الكمية المطلوبة من '{mat_name}' لوحدة واحدة: "))
            # ابحث عن المادة
            c.execute("SELECT id FROM raw_materials WHERE name = ?", (mat_name,))
            mat = c.fetchone()
            if not mat:
                print(f"❌ المادة '{mat_name}' غير موجودة! أضفها أولًا.")
                continue
            c.execute("INSERT INTO production_recipes (product_id, raw_material_id, quantity_needed) VALUES (?, ?, ?)",
                      (product_id, mat[0], qty))
        conn.commit()
        print("✅ تم إضافة المنتج ووصفة الإنتاج!")
    except Exception as e:
        print("❌ خطأ:", e)
    conn.close()

# === 3. تسجيل إنتاج ===
def record_production():
    product_name = input("اسم المنتج: ").strip()
    quantity = int(input("الكمية المنتجة: "))

    conn = get_connection()
    c = conn.cursor()

    # احصل على معلومات المنتج
    c.execute("SELECT id, name FROM products WHERE name = ?", (product_name,))
    product = c.fetchone()
    if not product:
        print("❌ المنتج غير موجود!")
        conn.close()
        return

    product_id = product[0]

    # احسب الكميات المطلوبة من الخامات
    c.execute("""
        SELECT rm.name, rm.id, rm.current_stock, pr.quantity_needed
        FROM production_recipes pr
        JOIN raw_materials rm ON pr.raw_material_id = rm.id
        WHERE pr.product_id = ?
    """, (product_id,))
    materials = c.fetchall()

    if not materials:
        print("❌ لا توجد وصفة إنتاج لهذا المنتج!")
        conn.close()
        return

    # تحقق من توفر الكميات
    for mat in materials:
        needed = mat[3] * quantity
        if needed > mat[2]:
            print(f"❌ لا يوجد ما يكفي من '{mat[0]}'. المطلوب: {needed}, المتاح: {mat[2]}")
            conn.close()
            return

    # خصم الخامات
    for mat in materials:
        needed = mat[3] * quantity
        c.execute("UPDATE raw_materials SET current_stock = current_stock - ? WHERE id = ?", (needed, mat[1]))

    # سجل الإنتاج
    c.execute("INSERT INTO production_logs (product_id, quantity_produced, date) VALUES (?, ?, ?)",
              (product_id, quantity, datetime.now().strftime("%Y-%m-%d")))

    conn.commit()
    print(f"✅ تم إنتاج {quantity} وحدة من '{product_name}' وخُصمت الخامات!")
    conn.close()

# === 4. إصدار فاتورة ===
def create_invoice():
    customer = input("اسم العميل: ").strip()
    items = []
    total = 0

    conn = get_connection()
    c = conn.cursor()

    while True:
        prod_name = input("اسم المنتج (أو 'تم' للانتهاء): ").strip()
        if prod_name.lower() in ['تم', 'done', 'finish']:
            break
        qty = int(input("الكمية: "))

        c.execute("SELECT id, selling_price FROM products WHERE name = ?", (prod_name,))
        prod = c.fetchone()
        if not prod:
            print("❌ منتج غير موجود!")
            continue

        price = prod[1]
        item_total = qty * price
        total += item_total
        items.append((prod[0], qty, price))

    if not items:
        print("❌ لم يتم إضافة أي منتجات!")
        conn.close()
        return

    paid = float(input(f"المبلغ المدفوع (من أصل {total}): ") or 0)
    if paid > total:
        paid = total

    # أدخل الفاتورة
    c.execute("INSERT INTO invoices (customer_name, total_amount, paid_amount, date) VALUES (?, ?, ?, ?)",
              (customer, total, paid, datetime.now().strftime("%Y-%m-%d")))
    invoice_id = c.lastrowid

    # أدخل التفاصيل
    for item in items:
        c.execute("INSERT INTO invoice_items (invoice_id, product_id, quantity, price_per_unit) VALUES (?, ?, ?, ?)",
                  (invoice_id, item[0], item[1], item[2]))

    conn.commit()
    print(f"✅ تم إصدار فاتورة بقيمة {total} ج.م. للمتبقي: {total - paid}")
    conn.close()

# === 5. عرض التقارير ===
def show_reports():
    conn = get_connection()
    c = conn.cursor()

    # الإيرادات (المدفوع)
    c.execute("SELECT SUM(paid_amount) FROM invoices")
    revenue = c.fetchone()[0] or 0

    # المصروفات (شراء خامات + أخرى)
    c.execute("SELECT SUM(amount) FROM expenses")
    manual_expenses = c.fetchone()[0] or 0

    # تكلفة الخامات المستهلكة (من الإنتاج)
    c.execute("""
        SELECT SUM(rm.cost_per_unit * pr.quantity_needed * pl.quantity_produced)
        FROM production_logs pl
        JOIN production_recipes pr ON pl.product_id = pr.product_id
        JOIN raw_materials rm ON pr.raw_material_id = rm.id
    """)
    material_cost = c.fetchone()[0] or 0

    total_expenses = manual_expenses + material_cost
    profit = revenue - total_expenses

    print("\n" + "="*50)
    print("📊 تقرير الأرباح والخسائر")
    print("="*50)
    print(f"إجمالي الإيرادات (المدفوعة): {revenue:.2f}")
    print(f"تكلفة الخامات المستهلكة: {material_cost:.2f}")
    print(f"مصروفات إضافية: {manual_expenses:.2f}")
    print(f"إجمالي المصروفات: {total_expenses:.2f}")
    print(f"{'-'*50}")
    print(f"الربح الصافي: {profit:.2f} {'🟢' if profit >= 0 else '🔴'}")
    print("="*50)

    # عرض الديون
    c.execute("SELECT customer_name, (total_amount - paid_amount) AS due FROM invoices WHERE due > 0")
    debts = c.fetchall()
    if debts:
        print("\n💳 الديون المستحقة:")
        for cust, due in debts:
            print(f" - {cust}: {due:.2f}")
    else:
        print("\n✅ لا توجد ديون حالياً.")

    conn.close()

# === القائمة الرئيسية ===
def main_menu():
    init_db()
    while True:
        print("\n" + "="*40)
        print("🏭 نظام إدارة مصنع الأحذية")
        print("="*40)
        print("1. إضافة مادة خام")
        print("2. إضافة منتج (مع وصفة إنتاج)")
        print("3. تسجيل عملية إنتاج")
        print("4. إصدار فاتورة بيع")
        print("5. عرض التقارير (أرباح/خسائر/ديون)")
        print("6. الخروج")
        choice = input("اختر رقم الخيار: ").strip()

        if choice == "1":
            add_raw_material()
        elif choice == "2":
            add_product()
        elif choice == "3":
            record_production()
        elif choice == "4":
            create_invoice()
        elif choice == "5":
            show_reports()
        elif choice == "6":
            print("👋 وداعًا!")
            break
        else:
            print("❌ خيار غير صحيح!")

if __name__ == "__main__":
    main_menu()
