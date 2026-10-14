The earlier code was fine on speed. With a handful of transactions there is nothing to optimize, so the real problem was that it read like advanced code: shortcuts a beginner wouldn't write and couldn't easily explain. I rewrote all 5 modules in plain style: simple `for` loops, `if`/`elif`, and short comments. **The contract is unchanged**, so you can still work in parallel and swap these in. I ran the full app again with every test case (bad inputs, delete, reports, restart, negative balance, empty or missing file, and a CSV saved from Excel), and it all passes.

If your teacher asks "explain this line," you should be able to answer for your own module, so read yours once before you push.

**Member 1: `menu.py`**
```python
# Member 1 - Menu
# This file only talks to transaction_service. All input() and print() are here.

import transaction_service as service


def show_menu():
    print()
    print("===== Personal Expense Tracker =====")
    print("1. Add transaction")
    print("2. View transactions")
    print("3. Delete transaction")
    print("4. Summary")
    print("5. Category report")
    print("6. Monthly report")
    print("7. Largest expense")
    print("8. Exit")


def show_transactions(transactions):
    if len(transactions) == 0:
        print("No transactions yet.")
        return
    print("No. | Date | Type | Category | Amount | Note")
    number = 1
    for t in transactions:
        print(number, "|", t["date"], "|", t["type"], "|", t["category"], "|", f"{t['amount']:.2f}", "|", t["note"])
        number = number + 1


def show_summary(data):
    print("Total income  :", f"{data['income']:.2f}")
    print("Total expenses:", f"{data['expenses']:.2f}")
    print("Balance       :", f"{data['balance']:.2f}")


def add_screen():
    date = input("Date (YYYY-MM-DD): ")
    type_ = input("Type (income/expense): ")
    category = input("Category: ")
    amount = input("Amount: ")
    note = input("Note (optional): ")
    ok, message = service.add_transaction(date, type_, category, amount, note)
    if ok:
        print(message)
    else:
        print("Error:", message)


def view_screen():
    show_transactions(service.get_all_transactions())


def delete_screen():
    transactions = service.get_all_transactions()
    show_transactions(transactions)
    if len(transactions) == 0:
        return
    number = input("Enter the number to delete: ")
    ok, message = service.delete_transaction(number)
    if ok:
        print(message)
    else:
        print("Error:", message)


def summary_screen():
    show_summary(service.summary())


def category_screen():
    report = service.category_report()

    print("--- Expenses by category ---")
    if len(report["expense"]) == 0:
        print("None")
    for category in report["expense"]:
        print(category, ":", f"{report['expense'][category]:.2f}")

    print("--- Income by category ---")
    if len(report["income"]) == 0:
        print("None")
    for category in report["income"]:
        print(category, ":", f"{report['income'][category]:.2f}")


def monthly_screen():
    month = input("Month (YYYY-MM): ")
    ok, result = service.monthly_report(month)
    if not ok:
        print("Error:", result)
        return
    print("--- Summary for", month.strip(), "---")
    show_summary(result)


def largest_screen():
    t = service.largest_expense()
    if t is None:
        print("No expenses yet.")
    else:
        print("Largest expense:", f"{t['amount']:.2f}", "on", t["category"], "(" + t["date"] + ")", t["note"])


def run():
    while True:
        show_menu()
        choice = input("Enter choice: ").strip()

        if choice == "1":
            add_screen()
        elif choice == "2":
            view_screen()
        elif choice == "3":
            delete_screen()
        elif choice == "4":
            summary_screen()
        elif choice == "5":
            category_screen()
        elif choice == "6":
            monthly_screen()
        elif choice == "7":
            largest_screen()
        elif choice == "8":
            print("Goodbye!")
            break
        else:
            print("Invalid choice. Enter a number from 1 to 8.")
```

**Member 1: `main.py`**
```python
# Member 1 - Start the app with: python main.py

import menu

menu.run()
```

---

**Member 2: `validator.py`**
```python
# Member 2 - Validation
# Every function returns (True, clean_value) or (False, error_message)

import math
from datetime import datetime


def validate_date(text):
    text = text.strip()
    try:
        date = datetime.strptime(text, "%Y-%m-%d")
    except ValueError:
        return False, "Invalid date. Use YYYY-MM-DD, for example 2026-10-14."
    # this rejects short forms like 2026-1-5
    if len(text) != 10:
        return False, "Invalid date. Use YYYY-MM-DD, for example 2026-10-14."
    if date > datetime.today():
        return False, "Date cannot be in the future."
    return True, text


def validate_type(text):
    text = text.strip().lower()
    if text == "income" or text == "expense":
        return True, text
    return False, "Type must be income or expense."


def validate_category(text):
    text = text.strip().lower()
    if text == "":
        return False, "Category cannot be empty."
    return True, text


def validate_amount(text):
    try:
        amount = float(text.strip())
    except ValueError:
        return False, "Amount must be a number, for example 250 or 99.50."
    # this rejects nan and inf
    if not math.isfinite(amount):
        return False, "Amount must be a valid number."
    amount = round(amount, 2)
    if amount <= 0:
        return False, "Amount must be greater than 0."
    return True, amount


def validate_month(text):
    text = text.strip()
    try:
        datetime.strptime(text, "%Y-%m")
    except ValueError:
        return False, "Invalid month. Use YYYY-MM, for example 2026-10."
    if len(text) != 7:
        return False, "Invalid month. Use YYYY-MM, for example 2026-10."
    return True, text
```

---

**Member 3: `storage.py`** (already includes the Excel fix)
```python
# Member 3 - Storage (reads and writes the CSV file)

import csv
import os

# the csv file is saved in the same folder as this code
FILE_NAME = os.path.join(os.path.dirname(os.path.abspath(__file__)), "transactions.csv")
FIELDS = ["date", "type", "category", "amount", "note"]


def load_transactions():
    transactions = []
    if not os.path.exists(FILE_NAME):
        return transactions

    with open(FILE_NAME, "r", newline="", encoding="utf-8-sig") as file:
        reader = csv.DictReader(file)
        for row in reader:
            try:
                transaction = {
                    "date": row["date"],
                    "type": row["type"],
                    "category": row["category"],
                    "amount": float(row["amount"]),
                    "note": row["note"] or "",
                }
                transactions.append(transaction)
            except (KeyError, ValueError, TypeError):
                pass  # skip a broken row instead of crashing
    return transactions


def save_transactions(transactions):
    # returns True if saved, False if the file could not be written
    try:
        with open(FILE_NAME, "w", newline="", encoding="utf-8") as file:
            writer = csv.DictWriter(file, fieldnames=FIELDS)
            writer.writeheader()
            for transaction in transactions:
                writer.writerow(transaction)
        return True
    except OSError:
        return False
```

---

**Member 4: `transaction_service.py`**
```python
# Member 4 - Transaction service
# This is the only file that talks to validator, storage and reports.

import reports
import storage
import validator

SAVE_ERROR = "Could not save transactions.csv. Is it open in Excel?"


def balance_warning(transactions):
    balance = reports.get_summary(transactions)["balance"]
    if balance < 0:
        return "\nWARNING: your balance is negative (" + str(balance) + ")."
    return ""


def add_transaction(date, type_, category, amount, note=""):
    ok, result = validator.validate_date(date)
    if not ok:
        return False, result
    date = result

    ok, result = validator.validate_type(type_)
    if not ok:
        return False, result
    type_ = result

    ok, result = validator.validate_category(category)
    if not ok:
        return False, result
    category = result

    ok, result = validator.validate_amount(amount)
    if not ok:
        return False, result
    amount = result

    transactions = storage.load_transactions()
    new_transaction = {
        "date": date,
        "type": type_,
        "category": category,
        "amount": amount,
        "note": note.strip(),
    }
    transactions.append(new_transaction)

    if not storage.save_transactions(transactions):
        return False, SAVE_ERROR
    return True, "Transaction added." + balance_warning(transactions)


def get_all_transactions():
    return storage.load_transactions()


def delete_transaction(index):
    # index is the number shown in the list (it starts from 1)
    try:
        index = int(index.strip())
    except ValueError:
        return False, "Please enter a valid number."

    transactions = storage.load_transactions()
    if index < 1 or index > len(transactions):
        return False, "No transaction with number " + str(index) + "."

    removed = transactions.pop(index - 1)
    if not storage.save_transactions(transactions):
        return False, SAVE_ERROR
    message = "Deleted: " + removed["date"] + " " + removed["category"] + " " + str(removed["amount"])
    return True, message + balance_warning(transactions)


def summary():
    return reports.get_summary(storage.load_transactions())


def category_report():
    transactions = storage.load_transactions()
    return {
        "expense": reports.get_category_totals(transactions, "expense"),
        "income": reports.get_category_totals(transactions, "income"),
    }


def monthly_report(month):
    # returns (True, summary) or (False, error message)
    ok, result = validator.validate_month(month)
    if not ok:
        return False, result
    return True, reports.get_monthly_summary(storage.load_transactions(), result)


def largest_expense():
    return reports.get_largest_expense(storage.load_transactions())
```

---

**Member 5: `reports.py`**
```python
# Member 5 - Reports
# These functions only calculate. No input(), no print(), no files.


def get_summary(transactions):
    income = 0
    expenses = 0
    for t in transactions:
        if t["type"] == "income":
            income = income + t["amount"]
        elif t["type"] == "expense":
            expenses = expenses + t["amount"]
    balance = income - expenses
    return {
        "income": round(income, 2),
        "expenses": round(expenses, 2),
        "balance": round(balance, 2),
    }


def get_category_totals(transactions, type_="expense"):
    totals = {}
    for t in transactions:
        if t["type"] == type_:
            category = t["category"]
            if category in totals:
                totals[category] = totals[category] + t["amount"]
            else:
                totals[category] = t["amount"]
    for category in totals:
        totals[category] = round(totals[category], 2)
    return totals


def get_monthly_summary(transactions, month):
    # month looks like 2026-10, the first 7 letters of a date
    month_transactions = []
    for t in transactions:
        if t["date"][:7] == month:
            month_transactions.append(t)
    return get_summary(month_transactions)


def get_largest_expense(transactions):
    largest = None
    for t in transactions:
        if t["type"] == "expense":
            if largest is None or t["amount"] > largest["amount"]:
                largest = t
    return largest
```

---

**What I simplified:**
- Removed the shortcut-style code (`lambda`, `max(..., key=...)`, list tricks) and used plain loops.
- Replaced the menu's dictionary of functions with a simple `if/elif` chain.
- Replaced the table spacing codes with simple `|` separators.
- Shortened the long docstrings to short comments.

Everything else (function names, inputs, return values) is identical, so nothing breaks between modules.
