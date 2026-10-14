I ran all five modules together end to end and tested these cases:
- Bad inputs (future date, wrong date format, letters or negative amount, wrong type, empty category).
- Save and reload, then a restart with the records intact.
- Delete.
- All the reports.
- The negative balance warning.

Everything worked. Each file below goes in the repo root, and each member copies only their own.

**Small additions to the contract (tell your team):**
- `validate_month(text)` was added to the validator, because the monthly report needs it.
- `save_transactions()` returns `True` or `False`, so the app doesn't crash if the CSV is open in Excel.
- `delete_transaction(index)` takes the number shown in the list (starts at 1).
- `monthly_report(month)` returns `(True, dict)` or `(False, error)`.
- `get_category_totals` has an optional second argument, `"expense"` or `"income"`.

---

**Member 1: `menu.py`**
```python
"""Member 1 - Menu / UI. Talks ONLY to transaction_service. Owns all input() and print()."""
import transaction_service as service

MENU = """
===== Personal Expense Tracker =====
1. Add transaction
2. View transactions
3. Delete transaction
4. Summary (income, expenses, balance)
5. Category report
6. Monthly report
7. Largest expense
8. Exit"""


def show_transactions(transactions):
    if not transactions:
        print("No transactions yet.")
        return
    print(f"{'No.':<5}{'Date':<12}{'Type':<9}{'Category':<15}{'Amount':>10}  Note")
    for i, t in enumerate(transactions, start=1):
        print(f"{i:<5}{t['date']:<12}{t['type']:<9}{t['category']:<15}{t['amount']:>10.2f}  {t['note']}")


def show_summary(data):
    print(f"Total income   : {data['income']:.2f}")
    print(f"Total expenses : {data['expenses']:.2f}")
    print(f"Balance        : {data['balance']:.2f}")


def add_screen():
    date = input("Date (YYYY-MM-DD): ")
    type_ = input("Type (income/expense): ")
    category = input("Category: ")
    amount = input("Amount: ")
    note = input("Note (optional): ")
    ok, message = service.add_transaction(date, type_, category, amount, note)
    print(message if ok else f"Error: {message}")


def view_screen():
    show_transactions(service.get_all_transactions())


def delete_screen():
    transactions = service.get_all_transactions()
    show_transactions(transactions)
    if not transactions:
        return
    number = input("Enter the number to delete: ")
    ok, message = service.delete_transaction(number)
    print(message if ok else f"Error: {message}")


def summary_screen():
    show_summary(service.summary())


def category_screen():
    report = service.category_report()
    for kind in ("expense", "income"):
        print(f"--- {kind.capitalize()} by category ---")
        if not report[kind]:
            print("(none)")
        for category, total in report[kind].items():
            print(f"{category:<15}{total:>10.2f}")


def monthly_screen():
    month = input("Month (YYYY-MM): ")
    ok, result = service.monthly_report(month)
    if not ok:
        print(f"Error: {result}")
        return
    print(f"--- Summary for {month.strip()} ---")
    show_summary(result)


def largest_screen():
    t = service.largest_expense()
    if t is None:
        print("No expenses recorded yet.")
    else:
        print(f"Largest expense: {t['amount']:.2f} on {t['category']} ({t['date']}) {t['note']}")


def run():
    actions = {
        "1": add_screen, "2": view_screen, "3": delete_screen, "4": summary_screen,
        "5": category_screen, "6": monthly_screen, "7": largest_screen,
    }
    while True:
        print(MENU)
        choice = input("Enter choice: ").strip()
        if choice == "8":
            print("Goodbye!")
            break
        action = actions.get(choice)
        if action:
            action()
        else:
            print("Invalid choice. Enter a number from 1 to 8.")
```

**Member 1: `main.py`**
```python
"""Member 1 - Entry point. Run with: python main.py"""
import menu

if __name__ == "__main__":
    try:
        menu.run()
    except (KeyboardInterrupt, EOFError):
        print("\nGoodbye!")
```

---

**Member 2: `validator.py`**
```python
"""Member 2 - Validation.
Every function returns (True, clean_value) or (False, error_message)."""
import math
from datetime import datetime


def validate_date(text):
    text = str(text).strip()
    try:
        parsed = datetime.strptime(text, "%Y-%m-%d")
    except ValueError:
        return False, "Invalid date. Use YYYY-MM-DD, e.g. 2026-10-14."
    if len(text) != 10:
        return False, "Invalid date. Use YYYY-MM-DD, e.g. 2026-10-14."
    if parsed.date() > datetime.now().date():
        return False, "Date cannot be in the future."
    return True, text


def validate_type(text):
    value = str(text).strip().lower()
    if value not in ("income", "expense"):
        return False, "Type must be 'income' or 'expense'."
    return True, value


def validate_category(text):
    value = str(text).strip().lower()
    if not value:
        return False, "Category cannot be empty."
    return True, value


def validate_amount(text):
    try:
        value = float(str(text).strip())
    except ValueError:
        return False, "Amount must be a number, e.g. 250 or 99.50."
    if not math.isfinite(value):
        return False, "Amount must be a valid number."
    value = round(value, 2)
    if value <= 0:
        return False, "Amount must be a positive number."
    return True, value


def validate_month(text):
    text = str(text).strip()
    try:
        datetime.strptime(text, "%Y-%m")
    except ValueError:
        return False, "Invalid month. Use YYYY-MM, e.g. 2026-10."
    if len(text) != 7:
        return False, "Invalid month. Use YYYY-MM, e.g. 2026-10."
    return True, text
```

---

**Member 3: `storage.py`**
```python
"""Member 3 - Storage. Reads and writes transactions.csv."""
import csv
import os

FILE_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "transactions.csv")
FIELDS = ["date", "type", "category", "amount", "note"]


def load_transactions():
    """Return a list of transaction dicts. Empty list if the file is missing or empty."""
    if not os.path.exists(FILE_PATH):
        return []
    transactions = []
    with open(FILE_PATH, newline="", encoding="utf-8") as f:
        for row in csv.DictReader(f):
            try:
                transactions.append({
                    "date": row["date"],
                    "type": row["type"],
                    "category": row["category"],
                    "amount": float(row["amount"]),
                    "note": row.get("note") or "",
                })
            except (KeyError, ValueError, TypeError):
                continue  # skip damaged rows instead of crashing
    return transactions


def save_transactions(transactions):
    """Write the whole list to the CSV. Returns True on success, False if it fails."""
    try:
        with open(FILE_PATH, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=FIELDS)
            writer.writeheader()
            writer.writerows(transactions)
        return True
    except OSError:
        return False
```

---

**Member 4: `transaction_service.py`**
```python
"""Member 4 - Transaction service. The only module that talks to validator, storage and reports."""
import reports
import storage
import validator

SAVE_ERROR = "Could not save to transactions.csv. Is it open in another program?"


def _balance_warning(transactions):
    balance = reports.get_summary(transactions)["balance"]
    if balance < 0:
        return f"\nWARNING: your balance is negative ({balance:.2f})."
    return ""


def add_transaction(date, type_, category, amount, note=""):
    """Takes raw text from the menu. Returns (True/False, message)."""
    checks = [
        validator.validate_date(date),
        validator.validate_type(type_),
        validator.validate_category(category),
        validator.validate_amount(amount),
    ]
    for ok, value in checks:
        if not ok:
            return False, value
    date, type_, category, amount = (value for _, value in checks)

    transactions = storage.load_transactions()
    transactions.append({
        "date": date,
        "type": type_,
        "category": category,
        "amount": amount,
        "note": str(note).strip(),
    })
    if not storage.save_transactions(transactions):
        return False, SAVE_ERROR
    return True, "Transaction added." + _balance_warning(transactions)


def get_all_transactions():
    return storage.load_transactions()


def delete_transaction(index):
    """index is the number shown in the list (starts at 1). Returns (True/False, message)."""
    try:
        index = int(str(index).strip())
    except ValueError:
        return False, "Please enter a valid number."
    transactions = storage.load_transactions()
    if not 1 <= index <= len(transactions):
        return False, f"No transaction number {index}."
    removed = transactions.pop(index - 1)
    if not storage.save_transactions(transactions):
        return False, SAVE_ERROR
    message = f"Deleted: {removed['date']} {removed['category']} {removed['amount']:.2f}"
    return True, message + _balance_warning(transactions)


def summary():
    return reports.get_summary(storage.load_transactions())


def category_report():
    transactions = storage.load_transactions()
    return {
        "expense": reports.get_category_totals(transactions, "expense"),
        "income": reports.get_category_totals(transactions, "income"),
    }


def monthly_report(month):
    """Returns (True, summary_dict) or (False, error_message)."""
    ok, value = validator.validate_month(month)
    if not ok:
        return False, value
    return True, reports.get_monthly_summary(storage.load_transactions(), value)


def largest_expense():
    return reports.get_largest_expense(storage.load_transactions())
```

---

**Member 5: `reports.py`**
```python
"""Member 5 - Reports. Pure functions: take a list, return results. No input(), print() or files."""


def get_summary(transactions):
    income = sum(t["amount"] for t in transactions if t["type"] == "income")
    expenses = sum(t["amount"] for t in transactions if t["type"] == "expense")
    return {
        "income": round(income, 2),
        "expenses": round(expenses, 2),
        "balance": round(income - expenses, 2),
    }


def get_category_totals(transactions, type_="expense"):
    totals = {}
    for t in transactions:
        if t["type"] == type_:
            totals[t["category"]] = round(totals.get(t["category"], 0) + t["amount"], 2)
    return totals


def get_monthly_summary(transactions, month):
    """month looks like '2026-10'."""
    in_month = [t for t in transactions if t["date"].startswith(month)]
    return get_summary(in_month)


def get_largest_expense(transactions):
    expenses = [t for t in transactions if t["type"] == "expense"]
    if not expenses:
        return None
    return max(expenses, key=lambda t: t["amount"])
```

---

**Run it:** put all 6 files in one folder, open a terminal there and type `python main.py`.

**Before you push:** add a `.gitignore` file containing `transactions.csv` and `__pycache__/`. Otherwise everyone's personal test data will clash in Git.

**Demo for the "Definition of Done":** add 5 transactions, view the list, delete one, show the category report and the monthly report (like `2026-10`), then exit with option 8, run `python main.py` again, and view the list to show the records are still there.

Want a `README.md` and the `.gitignore` next?
