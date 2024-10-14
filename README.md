# Inventory_System_Managements-
import sqlite3
import tkinter as tk
from tkinter import messagebox, ttk
from tkinter import scrolledtext
import matplotlib.pyplot as plt

# Database setup
def setup_database():
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS products (
        product_id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        price REAL NOT NULL,
        quantity INTEGER NOT NULL,
        min_stock INTEGER NOT NULL
    )
    """)
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS sales (
        sale_id INTEGER PRIMARY KEY AUTOINCREMENT,
        product_id INTEGER NOT NULL,
        quantity INTEGER NOT NULL,
        FOREIGN KEY (product_id) REFERENCES products (product_id)
    )
    """)
    connection.commit()
    connection.close()

# Add product
def add_product(name, price, quantity, min_stock):
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("INSERT INTO products (name, price, quantity, min_stock) VALUES (?, ?, ?, ?)", (name, price, quantity, min_stock))
    connection.commit()
    connection.close()
    update_product_list()
    check_stock_levels()

# Edit product
def edit_product(product_id, name, price, quantity, min_stock):
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("UPDATE products SET name=?, price=?, quantity=?, min_stock=? WHERE product_id=?", (name, price, quantity, min_stock, product_id))
    connection.commit()
    connection.close()
    update_product_list()
    check_stock_levels()

# Delete product
def delete_product(product_id):
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("DELETE FROM products WHERE product_id=?", (product_id,))
    connection.commit()
    connection.close()
    update_product_list()
    check_stock_levels()

# Sell product
def sell_product(product_id, quantity):
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("SELECT quantity FROM products WHERE product_id=?", (product_id,))
    result = cursor.fetchone()
    if result and result[0] >= quantity:
        cursor.execute("UPDATE products SET quantity=quantity-? WHERE product_id=?", (quantity, product_id))
        cursor.execute("INSERT INTO sales (product_id, quantity) VALUES (?, ?)", (product_id, quantity))
        connection.commit()
    else:
        messagebox.showerror("Error", "Not enough stock to sell")
    connection.close()
    update_product_list()
    check_stock_levels()

# Check stock levels
def check_stock_levels():
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("SELECT name, quantity, min_stock FROM products WHERE quantity < min_stock")
    low_stock_items = cursor.fetchall()
    if low_stock_items:
        alert_message = "Low Stock Alert:\n"
        for item in low_stock_items:
            alert_message += f"Product: {item[0]}, Current Quantity: {item[1]}, Minimum Stock: {item[2]}\n"
        messagebox.showwarning("Low Stock Alert", alert_message)
    connection.close()

# Generate sales summary and pie chart
def generate_sales_summary():
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("""
    SELECT products.name, SUM(sales.quantity) AS total_quantity, products.price
    FROM sales
    JOIN products ON sales.product_id = products.product_id
    GROUP BY products.product_id
    """)
    sales_data = cursor.fetchall()
    
    report = "Sales Summary:\n\n"
    total_sales = 0
    product_names = []
    quantities_sold = []
    
    for item in sales_data:
        product_name, total_quantity, price = item
        revenue = total_quantity * price
        total_sales += revenue
        report += f"Product: {product_name}, Quantity Sold: {total_quantity}, Revenue: ${revenue:.2f}\n"
        product_names.append(product_name)
        quantities_sold.append(total_quantity)
    
    report += f"\nTotal Revenue: ${total_sales:.2f}"
    
    # Display report in a new window
    report_window = tk.Toplevel()
    report_window.title("Sales Summary Report")
    report_window.geometry("400x300")
    
    text_area = scrolledtext.ScrolledText(report_window, wrap=tk.WORD, font=('Arial', 12))
    text_area.insert(tk.END, report)
    text_area.configure(state='disabled')  # Make it read-only
    text_area.pack(expand=True, fill='both')

    # Create pie chart
    plt.figure(figsize=(6, 6))
    plt.pie(quantities_sold, labels=product_names, autopct='%1.1f%%', startangle=90)
    plt.title('Sales Distribution')
    plt.axis('equal')  # Equal aspect ratio ensures that pie chart is circular
    plt.show()

    connection.close()

# Update product list
def update_product_list():
    for row in product_tree.get_children():
        product_tree.delete(row)
    connection = sqlite3.connect("inventory.db")
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM products")
    for row in cursor.fetchall():
        product_tree.insert("", tk.END, values=row)
    connection.close()

# Create the main application window
def inventory_gui():
    global product_tree

    root = tk.Tk()
    root.title("Inventory Management System")
    root.geometry("800x600")
    root.configure(bg='black')

    # Input fields
    tk.Label(root, text="Product Name", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    product_name = tk.Entry(root, font=('Arial', 14))
    product_name.pack(pady=5)

    tk.Label(root, text="Price", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    product_price = tk.Entry(root, font=('Arial', 14))
    product_price.pack(pady=5)

    tk.Label(root, text="Quantity", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    product_quantity = tk.Entry(root, font=('Arial', 14))
    product_quantity.pack(pady=5)

    tk.Label(root, text="Min Stock", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    product_min_stock = tk.Entry(root, font=('Arial', 14))
    product_min_stock.pack(pady=5)

    # Buttons for product management
    tk.Button(root, text="Add Product", command=lambda: add_product(product_name.get(), float(product_price.get()), int(product_quantity.get()), int(product_min_stock.get())), bg='green', fg='white', font=('Arial', 14)).pack(pady=5)
    tk.Button(root, text="Edit Product", command=lambda: edit_product(product_tree.item(product_tree.selection())['values'][0], product_name.get(), float(product_price.get()), int(product_quantity.get()), int(product_min_stock.get())), bg='yellow', fg='black', font=('Arial', 14)).pack(pady=5)
    tk.Button(root, text="Delete Product", command=lambda: delete_product(product_tree.item(product_tree.selection())['values'][0]), bg='red', fg='white', font=('Arial', 14)).pack(pady=5)

    # Sell product section
    sell_frame = tk.Frame(root, bg='black')
    sell_frame.pack(pady=(20, 10))

    tk.Label(sell_frame, text="Sell Product ID", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    sell_product_id = tk.Entry(sell_frame, font=('Arial', 14))
    sell_product_id.pack(pady=5)

    tk.Label(sell_frame, text="Sell Quantity", bg='black', fg='white', font=('Arial', 14)).pack(pady=5)
    sell_quantity = tk.Entry(sell_frame, font=('Arial', 14))
    sell_quantity.pack(pady=5)

    tk.Button(sell_frame, text="Sell Product", command=lambda: sell_product(int(sell_product_id.get()), int(sell_quantity.get())), bg='orange', fg='black', font=('Arial', 14)).pack(pady=5)

    # Sales Summary button
    tk.Button(root, text="Generate Sales Summary", command=generate_sales_summary, bg='blue', fg='white', font=('Arial', 14)).pack(pady=5)

    # Product treeview
    product_tree = ttk.Treeview(root, columns=("ID", "Name", "Price", "Quantity", "Min Stock"), show='headings')
    product_tree.heading("ID", text="ID")
    product_tree.heading("Name", text="Name")
    product_tree.heading("Price", text="Price")
    product_tree.heading("Quantity", text="Quantity")
    product_tree.heading("Min Stock", text="Min Stock")
    product_tree.pack(pady=(1,2), fill=tk.BOTH, expand=True)  # Increase space below the product list

    update_product_list()

    root.mainloop()

setup_database()
inventory_gui()
