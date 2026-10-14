
Your output shows that no commit was ever made, so there was nothing new to push. In the log, your branch, origin/<your branch>, origin/main and main all point to the same commit, “step 0 initial commit”. The push didn’t fail. It just created a remote branch identical to main.

Your remote URL is correct and you’re on a feature branch, so those aren’t the problem. The missing piece is the commit.

Find out why the commit didn’t happen

Run this:

powershell
git status
What it shows	Cause	Fix
Your files under “Changes not staged for commit” or “Untracked files”	git add or git commit never ran, or git add was run in another folder	Run the commands below
nothing to commit, working tree clean, but you changed code	The edits aren’t on disk in this folder. Either VS Code hasn’t saved them (press Ctrl+S) or you edited a copy of the project in a different folder	Check that VS Code has the cloned TrackitUP folder open, then save and run git status again
Your file isn’t listed, though you know it changed	.gitignore is excluding it	git check-ignore -v <your file> shows which rule matches
Then add, commit and push

Use your own file names:

powershell
git add transaction_service.py test_service.py test_integration.py
git commit -m "feat: add transaction service and tests"
git log --oneline -3
git push

Read what git commit prints. If it says Author identity unknown or Please tell me who you are, set your name and email once, then commit again:

powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
How to confirm it worked
git log --oneline -3 should now show your new commit on top, above “step 0 initial commit”.
On GitHub, open your branch from the branch dropdown. Your files should be there, and a “Compare & pull request” banner should appear. Click it to open the PR.

If git status shows something different from the table, paste its full output and I’ll read it.

Pasted text.txt
TXT

so this is an entire code base that we are running so basically it's run successfully in our terminal so now what's remaining is that we have to connect an easy to understand simple ui and front end in html javascript format to our current completed backend so analyze this end to end code base followed by its contract api design as well as all the components mentioned in this code base don't add anything by yourself update or assume anything by yourself according to according to this uh, backend code provide me in detail front end ui code files so that i can you can successfully run in end to end manner this respective feature on my front end so starting from step 1 to step 0 for the color theme you can mix blue and white and green and all don't make it as an AI generated flop

I’ve read the code you attached. It’s the plain-function version, with transactions as dictionaries and a menu that uses input() and print(), so I’m building against exactly that and won’t change any of your five modules. I’ll rebuild your backend in a scratch folder, add the web front end and the connection it needs, and test it end to end before sending anything.

Writing the bridge server and the four front-end files now.

Claude’s response was interrupted.
Compacted conversation

So I don't want you to create an entire code base folder. Instead, only give me front end code in our chat itself in a way that I can copy and paste in my current working code base that I have shared with you earlier. And it should get easily connected with all the APIs and modules present in my current code base that is uh, it mainly consists of an five to six module. It should be connected seamlessly and maintain continuity from start to end.

Backend bridge and all API endpoints test clean. I’m now running the UI in a headless browser to check every screen before I paste the code here in chat.

Your front end is built and tested. I added no new folder; all the code is below, ready to copy.

I ran your backend modules unchanged behind it and checked all 7 API routes with good and bad input. I also drove all 4 screens in a headless browser, including the negative-balance warning, delete confirmation, an empty month, and a phone-width layout. The one thing I could not test is the Google Fonts load, because my test machine is offline. If the fonts don’t load, the page uses Georgia and system fonts instead.

What goes where

You must add one new file, server.py. A browser cannot call Python functions directly, so something has to turn web requests into calls to your functions. server.py does only that. It uses Python’s built-in tools, so there is nothing to install. No existing module is edited.

your-project/
├── main.py                  (unchanged)
├── menu.py                  (unchanged)
├── validator.py             (unchanged)
├── storage.py               (unchanged)
├── transaction_service.py   (unchanged)
├── reports.py               (unchanged)
├── server.py                ← NEW
└── frontend/                ← NEW folder
    ├── index.html
    ├── css/styles.css
    └── js/api.js
    └── js/app.js
How to run
Step 0: Create the files below at the paths shown. Stop the terminal menu if it is open, because the CLI and the web server should not run at the same time.
Step 1: Run python server.py.
Step 2: Open http://127.0.0.1:8000 in your browser.

If port 8000 is busy, run python server.py 8001 and open that port instead.

File 1: server.py (same folder as main.py)
python
# server.py - lets the browser talk to your existing modules.
# It does NOT change menu.py, validator.py, storage.py, transaction_service.py or reports.py.
# Run:   python server.py
# Open:  http://127.0.0.1:8000

import json
import os
import sys
import threading
import traceback
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import parse_qs, unquote, urlparse

import transaction_service as service

HOST = "127.0.0.1"
DEFAULT_PORT = 8000
FRONTEND_FOLDER = os.path.join(os.path.dirname(os.path.abspath(__file__)), "frontend")
MAX_BODY_SIZE = 20000
CONTENT_TYPES = {
    ".html": "text/html; charset=utf-8",
    ".css": "text/css; charset=utf-8",
    ".js": "text/javascript; charset=utf-8",
}

# only one request may touch transactions.csv at a time
backend_lock = threading.Lock()


def text_value(data, key):
    # your service functions call .strip(), so every value must be text
    value = data.get(key, "")
    if value is None:
        return ""
    return str(value)


def numbered(transactions):
    # adds the same 1-based number that the terminal list shows
    result = []
    for position, t in enumerate(transactions, start=1):
        row = dict(t)
        row["number"] = position
        result.append(row)
    return result


def message_reply(ok, message, success_status=200):
    if ok:
        return success_status, {"ok": True, "message": message}
    return 400, {"ok": False, "message": message}


def call_api(method, path, query, data):
    # returns (status code, dictionary to send back as JSON)
    parts = [p for p in path.split("/") if p]
    resource = parts[1:]  # parts[0] is "api"

    if resource == ["transactions"]:
        if method == "GET":
            return 200, {"transactions": numbered(service.get_all_transactions())}
        if method == "POST":
            if not isinstance(data, dict):
                return 400, {"ok": False, "message": "Request body must be a JSON object."}
            ok, message = service.add_transaction(
                text_value(data, "date"),
                text_value(data, "type"),
                text_value(data, "category"),
                text_value(data, "amount"),
                text_value(data, "note"),
            )
            return message_reply(ok, message, 201)

    elif len(resource) == 2 and resource[0] == "transactions":
        if method == "DELETE":
            ok, message = service.delete_transaction(unquote(resource[1]))
            return message_reply(ok, message)

    elif resource == ["summary"]:
        if method == "GET":
            return 200, service.summary()

    elif resource == ["reports", "categories"]:
        if method == "GET":
            return 200, service.category_report()

    elif resource == ["reports", "monthly"]:
        if method == "GET":
            month = query.get("month", [""])[0]
            ok, result = service.monthly_report(month)
            if ok:
                return 200, {"ok": True, "month": month.strip(), "summary": result}
            return 400, {"ok": False, "message": result}

    elif resource == ["reports", "largest-expense"]:
        if method == "GET":
            return 200, {"largest_expense": service.largest_expense()}

    else:
        return 404, {"ok": False, "message": "Not found."}

    return 405, {"ok": False, "message": "This address does not allow " + method + "."}


class Handler(BaseHTTPRequestHandler):
    def send_bytes(self, status, content_type, body):
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(body)

    def send_json(self, status, payload):
        body = json.dumps(payload).encode("utf-8")
        self.send_bytes(status, "application/json; charset=utf-8", body)

    def read_json_body(self):
        # returns (data, error message)
        try:
            length = int(self.headers.get("Content-Length", "0"))
        except ValueError:
            return None, "Invalid request."
        if length > MAX_BODY_SIZE:
            return None, "Request is too large."
        if length == 0:
            return {}, None
        try:
            return json.loads(self.rfile.read(length).decode("utf-8")), None
        except (ValueError, UnicodeDecodeError):
            return None, "Request body must be valid JSON."

    def serve_static(self, path):
        if path == "/":
            path = "/index.html"
        root = os.path.realpath(FRONTEND_FOLDER)
        full = os.path.realpath(os.path.join(root, unquote(path).lstrip("/")))
        extension = os.path.splitext(full)[1].lower()
        if not full.startswith(root + os.sep) or not os.path.isfile(full) or extension not in CONTENT_TYPES:
            self.send_json(404, {"ok": False, "message": "Not found."})
            return
        with open(full, "rb") as file:
            self.send_bytes(200, CONTENT_TYPES[extension], file.read())

    def handle_request(self, method):
        url = urlparse(self.path)
        if url.path == "/api" or url.path.startswith("/api/"):
            data = None
            if method == "POST":
                data, error = self.read_json_body()
                if error:
                    self.send_json(400, {"ok": False, "message": error})
                    return
            try:
                with backend_lock:
                    status, payload = call_api(method, url.path, parse_qs(url.query), data)
            except Exception:
                traceback.print_exc()
                status = 500
                payload = {"ok": False, "message": "Something went wrong on the server. Check the terminal for details."}
            self.send_json(status, payload)
        elif method == "GET":
            self.serve_static(url.path)
        else:
            self.send_json(405, {"ok": False, "message": "Not allowed."})

    def do_GET(self):
        self.handle_request("GET")

    def do_POST(self):
        self.handle_request("POST")

    def do_DELETE(self):
        self.handle_request("DELETE")


def main():
    port = DEFAULT_PORT
    if len(sys.argv) > 1:
        try:
            port = int(sys.argv[1])
        except ValueError:
            print("Port must be a number, for example: python server.py 8001")
            return
    try:
        server = ThreadingHTTPServer((HOST, port), Handler)
    except OSError:
        print("Port " + str(port) + " is already in use. Close the other program or run: python server.py 8001")
        return
    server.daemon_threads = True
    print("TrackitUP is running at http://" + HOST + ":" + str(port))
    print("Press Ctrl+C to stop.")
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\nStopped.")


main()
File 2: frontend/index.html
html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>TrackitUP</title>
  <link rel="icon" href="data:,">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Public+Sans:wght@400;500;600&family=Zilla+Slab:wght@500;600&display=swap">
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <a class="skip-link" href="#main">Skip to content</a>

  <header class="topbar">
    <div class="topbar-inner">
      <p class="brand">TrackitUP</p>
      <nav aria-label="Main">
        <a data-route="overview" href="#/overview">Overview</a>
        <a data-route="transactions" href="#/transactions">Transactions</a>
        <a data-route="add" href="#/add">Add</a>
        <a data-route="reports" href="#/reports">Reports</a>
      </nav>
    </div>
  </header>

  <main id="main">
    <div id="notice" class="notice" role="status" aria-live="polite" hidden></div>

    <!-- Overview: summary(), largest_expense(), get_all_transactions() -->
    <section id="view-overview" class="sheet" hidden>
      <h1 id="title-overview" tabindex="-1">Overview</h1>

      <div id="figure-balance-box" class="balance-block">
        <p class="small-label">Balance</p>
        <p id="figure-balance" class="balance-figure">0.00</p>
        <p id="balance-note" class="balance-note"></p>
      </div>

      <div class="split">
        <div id="split-bar" class="split-bar" role="img" aria-label="No income or expenses yet">
          <span id="split-income" class="split-income"></span>
          <span id="split-expense" class="split-expense"></span>
        </div>
        <dl class="split-legend">
          <div><dt><span class="swatch swatch-income"></span>Income</dt><dd id="figure-income">0.00</dd></div>
          <div><dt><span class="swatch swatch-expense"></span>Expenses</dt><dd id="figure-expenses">0.00</dd></div>
        </dl>
      </div>

      <div class="two-col">
        <section aria-labelledby="heading-largest">
          <h2 id="heading-largest">Largest expense</h2>
          <div id="largest-expense"></div>
        </section>
        <section aria-labelledby="heading-recent">
          <h2 id="heading-recent">Latest entries</h2>
          <ul id="recent-list" class="ledger-list"></ul>
          <p class="more-link"><a href="#/transactions">See all transactions</a></p>
        </section>
      </div>
    </section>

    <!-- Transactions: get_all_transactions(), delete_transaction() -->
    <section id="view-transactions" class="sheet" hidden>
      <h1 id="title-transactions" tabindex="-1">Transactions</h1>
      <p class="intro">The number is the row's position in transactions.csv. When you delete a row, the rows below it move up by one.</p>

      <div id="transactions-empty" class="empty" hidden>
        <p>No transactions yet.</p>
        <p><a href="#/add">Add your first transaction</a></p>
      </div>

      <table id="transactions-table" class="ledger-table" hidden>
        <thead>
          <tr>
            <th scope="col">No.</th>
            <th scope="col">Date</th>
            <th scope="col">Type</th>
            <th scope="col">Category</th>
            <th scope="col">Note</th>
            <th scope="col" class="num">Amount</th>
            <th scope="col"><span class="visually-hidden">Actions</span></th>
          </tr>
        </thead>
        <tbody id="transactions-body"></tbody>
      </table>
    </section>

    <!-- Add: add_transaction() -->
    <section id="view-add" class="sheet" hidden>
      <h1 id="title-add" tabindex="-1">Add a transaction</h1>

      <form id="add-form" class="form-grid" novalidate>
        <div class="field">
          <label for="add-date">Date</label>
          <input id="add-date" name="date" type="date" required>
        </div>
        <div class="field">
          <label for="add-type">Type</label>
          <select id="add-type" name="type">
            <option value="expense">Expense</option>
            <option value="income">Income</option>
          </select>
        </div>
        <div class="field">
          <label for="add-category">Category</label>
          <input id="add-category" name="category" type="text" autocomplete="off" placeholder="food, rent, salary">
        </div>
        <div class="field">
          <label for="add-amount">Amount</label>
          <input id="add-amount" name="amount" type="text" inputmode="decimal" autocomplete="off" placeholder="250 or 99.50">
        </div>
        <div class="field field-wide">
          <label for="add-note">Note (optional)</label>
          <input id="add-note" name="note" type="text" autocomplete="off">
        </div>
        <div class="field-wide form-actions">
          <button id="add-submit" class="btn btn-green" type="submit">Add transaction</button>
        </div>
      </form>

      <p id="add-result" class="form-result" role="status" aria-live="polite" hidden></p>
    </section>

    <!-- Reports: category_report(), monthly_report() -->
    <section id="view-reports" class="sheet" hidden>
      <h1 id="title-reports" tabindex="-1">Reports</h1>

      <div class="two-col">
        <section aria-labelledby="heading-cat-expense">
          <h2 id="heading-cat-expense">Expenses by category</h2>
          <div id="category-expense"></div>
        </section>
        <section aria-labelledby="heading-cat-income">
          <h2 id="heading-cat-income">Income by category</h2>
          <div id="category-income"></div>
        </section>
      </div>

      <section class="month-block" aria-labelledby="heading-month">
        <h2 id="heading-month">Month summary</h2>
        <form id="month-form" class="month-form" novalidate>
          <div class="field">
            <label for="month-input">Month (YYYY-MM)</label>
            <input id="month-input" type="text" autocomplete="off" placeholder="2026-10">
          </div>
          <button class="btn btn-blue" type="submit">Show month</button>
        </form>
        <p id="month-error" class="field-error" role="alert" hidden></p>
        <div id="month-result"></div>
      </section>
    </section>
  </main>

  <script src="js/api.js"></script>
  <script src="js/app.js"></script>
</body>
</html>
File 3: frontend/js/api.js
javascript
// api.js - the only file that talks to server.py.
// Every function returns:
//   { ok: true, data: ... }                          when the server answered with success
//   { ok: false, unreachable: false, message: "" }   when the backend said no (message comes from your Python code)
//   { ok: false, unreachable: true,  message: "" }   when the server is not running

const Api = (function () {
  async function request(method, url, body) {
    let response;
    try {
      const options = { method: method, headers: {} };
      if (body !== undefined) {
        options.headers["Content-Type"] = "application/json";
        options.body = JSON.stringify(body);
      }
      response = await fetch(url, options);
    } catch (error) {
      return {
        ok: false,
        unreachable: true,
        message: "Cannot reach the server. Start it with: python server.py"
      };
    }

    let data = null;
    try {
      data = await response.json();
    } catch (error) {
      data = null;
    }

    if (response.ok) {
      return { ok: true, data: data };
    }
    const message = data && data.message ? data.message : "Something went wrong on the server.";
    return { ok: false, unreachable: false, message: message };
  }

  return {
    getTransactions: function () {
      return request("GET", "/api/transactions");
    },
    // fields = { date, type, category, amount, note }  (all text)
    addTransaction: function (fields) {
      return request("POST", "/api/transactions", fields);
    },
    deleteTransaction: function (number) {
      return request("DELETE", "/api/transactions/" + encodeURIComponent(number));
    },
    getSummary: function () {
      return request("GET", "/api/summary");
    },
    getCategoryReport: function () {
      return request("GET", "/api/reports/categories");
    },
    getMonthlyReport: function (month) {
      return request("GET", "/api/reports/monthly?month=" + encodeURIComponent(month));
    },
    getLargestExpense: function () {
      return request("GET", "/api/reports/largest-expense");
    }
  };
})();
File 4: frontend/js/app.js
javascript
// app.js - draws the pages and handles clicks.
// All numbers and messages come from your Python code through api.js.

(function () {
  "use strict";

  const ROUTES = ["overview", "transactions", "add", "reports"];
  const TITLES = {
    overview: "Overview",
    transactions: "Transactions",
    add: "Add a transaction",
    reports: "Reports"
  };

  // ---------- small helpers ----------

  function byId(id) {
    return document.getElementById(id);
  }

  function el(tag, className, text) {
    const node = document.createElement(tag);
    if (className) {
      node.className = className;
    }
    if (text !== undefined) {
      node.textContent = text;
    }
    return node;
  }

  function clear(node) {
    while (node.firstChild) {
      node.removeChild(node.firstChild);
    }
  }

  const moneyFormat = new Intl.NumberFormat(undefined, {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
  });

  function formatMoney(value) {
    return moneyFormat.format(value);
  }

  function signedMoney(transaction) {
    const sign = transaction.type === "income" ? "+" : "\u2212";
    return sign + formatMoney(transaction.amount);
  }

  function typeLabel(type) {
    return type === "income" ? "Income" : "Expense";
  }

  function todayText() {
    const now = new Date();
    const month = String(now.getMonth() + 1).padStart(2, "0");
    const day = String(now.getDate()).padStart(2, "0");
    return now.getFullYear() + "-" + month + "-" + day;
  }

  // ---------- notice bar ----------

  const noticeBox = byId("notice");

  function showNotice(message, kind) {
    noticeBox.textContent = message;
    noticeBox.className = "notice notice-" + kind;
    noticeBox.hidden = false;
  }

  function hideNotice() {
    noticeBox.hidden = true;
    noticeBox.textContent = "";
  }

  // returns true if every result worked, otherwise shows the problem and returns false
  function allOk(results) {
    for (let i = 0; i < results.length; i++) {
      if (!results[i].ok) {
        showNotice(results[i].message, "error");
        return false;
      }
    }
    return true;
  }

  // ---------- overview ----------

  async function loadOverview() {
    const results = await Promise.all([
      Api.getSummary(),
      Api.getLargestExpense(),
      Api.getTransactions()
    ]);
    if (!allOk(results)) {
      return;
    }
    const summary = results[0].data;
    const largest = results[1].data.largest_expense;
    const transactions = results[2].data.transactions;

    // balance
    byId("figure-balance").textContent = formatMoney(summary.balance);
    byId("figure-income").textContent = formatMoney(summary.income);
    byId("figure-expenses").textContent = formatMoney(summary.expenses);
    const negative = summary.balance < 0;
    byId("figure-balance-box").classList.toggle("is-negative", negative);
    byId("balance-note").textContent = negative
      ? "You have spent more than you earned."
      : "Income minus expenses.";

    // income / expense bar
    const total = summary.income + summary.expenses;
    const incomePart = total > 0 ? (summary.income / total) * 100 : 0;
    const expensePart = total > 0 ? (summary.expenses / total) * 100 : 0;
    byId("split-income").style.width = incomePart + "%";
    byId("split-expense").style.width = expensePart + "%";
    byId("split-bar").setAttribute(
      "aria-label",
      total > 0
        ? "Income " + formatMoney(summary.income) + ", expenses " + formatMoney(summary.expenses)
        : "No income or expenses yet"
    );

    // largest expense
    const largestBox = byId("largest-expense");
    clear(largestBox);
    if (largest === null) {
      largestBox.appendChild(el("p", "muted", "No expenses yet."));
    } else {
      largestBox.appendChild(el("p", "largest-amount", formatMoney(largest.amount)));
      largestBox.appendChild(el("p", "largest-category", largest.category));
      let detail = largest.date;
      if (largest.note) {
        detail = detail + " \u2013 " + largest.note;
      }
      largestBox.appendChild(el("p", "muted", detail));
    }

    // latest entries (last 5 rows in the file, newest first)
    const list = byId("recent-list");
    clear(list);
    if (transactions.length === 0) {
      list.appendChild(el("li", "muted", "Nothing added yet."));
    } else {
      const latest = transactions.slice(-5).reverse();
      latest.forEach(function (t) {
        const item = el("li", "ledger-row");
        item.appendChild(el("span", "ledger-date", t.date));
        const middle = el("span", "ledger-main");
        middle.appendChild(el("span", "category", t.category));
        if (t.note) {
          middle.appendChild(el("span", "muted note-line", t.note));
        }
        item.appendChild(middle);
        item.appendChild(el("span", "amount amount-" + t.type, signedMoney(t)));
        list.appendChild(item);
      });
    }
  }

  // ---------- transactions ----------

  function sameTransaction(a, b) {
    return (
      a.date === b.date &&
      a.type === b.type &&
      a.category === b.category &&
      a.amount === b.amount &&
      a.note === b.note
    );
  }

  async function deleteRow(transaction) {
    // the backend deletes by position, so first check the row is still where we think it is
    const fresh = await Api.getTransactions();
    if (!allOk([fresh])) {
      return;
    }
    const rows = fresh.data.transactions;
    const current = rows[transaction.number - 1];
    if (!current || !sameTransaction(current, transaction)) {
      showNotice("The list changed since it was loaded, so nothing was deleted. The list has been reloaded.", "error");
      renderTransactions(rows);
      return;
    }
    const result = await Api.deleteTransaction(transaction.number);
    if (result.ok) {
      showNotice(result.data.message, "success");
    } else {
      showNotice(result.message, "error");
    }
    loadTransactions();
  }

  function makeDeleteButton(transaction) {
    const button = el("button", "btn-delete", "Delete");
    button.type = "button";
    button.setAttribute("aria-label", "Delete transaction " + transaction.number);
    let timer = null;

    function disarm() {
      button.classList.remove("armed");
      button.textContent = "Delete";
      button.setAttribute("aria-label", "Delete transaction " + transaction.number);
    }

    button.addEventListener("click", function () {
      if (!button.classList.contains("armed")) {
        button.classList.add("armed");
        button.textContent = "Confirm delete";
        button.setAttribute("aria-label", "Confirm delete of transaction " + transaction.number);
        timer = setTimeout(disarm, 4000);
        return;
      }
      clearTimeout(timer);
      button.disabled = true;
      deleteRow(transaction);
    });
    return button;
  }

  function addCell(row, label, content, className) {
    const cell = el("td", className || "");
    cell.setAttribute("data-label", label);
    if (typeof content === "string") {
      cell.textContent = content;
    } else {
      cell.appendChild(content);
    }
    row.appendChild(cell);
  }

  function renderTransactions(transactions) {
    const body = byId("transactions-body");
    clear(body);
    const isEmpty = transactions.length === 0;
    byId("transactions-empty").hidden = !isEmpty;
    byId("transactions-table").hidden = isEmpty;

    transactions.forEach(function (t) {
      const row = el("tr");
      addCell(row, "No.", String(t.number), "col-number");
      addCell(row, "Date", t.date, "col-date");
      addCell(row, "Type", el("span", "tag tag-" + t.type, typeLabel(t.type)), "col-type");
      addCell(row, "Category", t.category, "col-category");
      addCell(row, "Note", t.note || "", "col-note");
      addCell(row, "Amount", signedMoney(t), "num amount amount-" + t.type);
      addCell(row, "", makeDeleteButton(t), "col-action");
      body.appendChild(row);
    });
  }

  async function loadTransactions() {
    const result = await Api.getTransactions();
    if (!allOk([result])) {
      return;
    }
    renderTransactions(result.data.transactions);
  }

  // ---------- add ----------

  function prepareAddForm() {
    const dateInput = byId("add-date");
    if (!dateInput.value) {
      dateInput.value = todayText();
    }
    byId("add-result").hidden = true;
  }

  function showAddResult(message, kind) {
    const box = byId("add-result");
    box.textContent = message;
    box.className = "form-result form-result-" + kind;
    box.hidden = false;
  }

  byId("add-form").addEventListener("submit", async function (event) {
    event.preventDefault();
    const button = byId("add-submit");
    button.disabled = true;

    const result = await Api.addTransaction({
      date: byId("add-date").value,
      type: byId("add-type").value,
      category: byId("add-category").value,
      amount: byId("add-amount").value,
      note: byId("add-note").value
    });

    button.disabled = false;
    if (result.ok) {
      showAddResult(result.data.message, "success");
      byId("add-category").value = "";
      byId("add-amount").value = "";
      byId("add-note").value = "";
      byId("add-category").focus();
    } else {
      showAddResult(result.message, "error");
    }
  });

  // ---------- reports ----------

  function renderCategoryTotals(container, totals, kind, emptyText) {
    clear(container);
    const names = Object.keys(totals).sort(function (a, b) {
      return totals[b] - totals[a];
    });
    if (names.length === 0) {
      container.appendChild(el("p", "muted", emptyText));
      return;
    }
    const biggest = totals[names[0]];
    names.forEach(function (name) {
      const row = el("div", "cat-row");
      const top = el("div", "cat-top");
      top.appendChild(el("span", "category", name));
      top.appendChild(el("span", "amount", formatMoney(totals[name])));
      row.appendChild(top);
      const track = el("div", "cat-track");
      const fill = el("div", "cat-fill cat-fill-" + kind);
      fill.style.width = (totals[name] / biggest) * 100 + "%";
      track.appendChild(fill);
      row.appendChild(track);
      container.appendChild(row);
    });
  }

  async function loadReports() {
    const result = await Api.getCategoryReport();
    if (!allOk([result])) {
      return;
    }
    renderCategoryTotals(byId("category-expense"), result.data.expense, "expense", "No expenses yet.");
    renderCategoryTotals(byId("category-income"), result.data.income, "income", "No income yet.");
    const monthInput = byId("month-input");
    if (!monthInput.value) {
      monthInput.value = todayText().slice(0, 7);
    }
  }

  function monthFigure(label, value, className) {
    const box = el("div", "month-figure");
    box.appendChild(el("dt", "", label));
    box.appendChild(el("dd", className || "", formatMoney(value)));
    return box;
  }

  byId("month-form").addEventListener("submit", async function (event) {
    event.preventDefault();
    const errorBox = byId("month-error");
    const resultBox = byId("month-result");
    errorBox.hidden = true;
    clear(resultBox);

    const result = await Api.getMonthlyReport(byId("month-input").value);
    if (!result.ok) {
      errorBox.textContent = result.message;
      errorBox.hidden = false;
      return;
    }
    const summary = result.data.summary;
    resultBox.appendChild(el("h3", "", result.data.month));
    if (summary.income === 0 && summary.expenses === 0) {
      resultBox.appendChild(el("p", "muted", "No transactions in this month."));
      return;
    }
    const list = el("dl", "month-figures");
    list.appendChild(monthFigure("Income", summary.income, "amount-income"));
    list.appendChild(monthFigure("Expenses", summary.expenses, "amount-expense"));
    list.appendChild(monthFigure("Balance", summary.balance, summary.balance < 0 ? "amount-negative" : ""));
    resultBox.appendChild(list);
  });

  // ---------- routing ----------

  const loaders = {
    overview: loadOverview,
    transactions: loadTransactions,
    add: prepareAddForm,
    reports: loadReports
  };

  function currentRoute() {
    const name = window.location.hash.replace("#/", "");
    return ROUTES.indexOf(name) >= 0 ? name : "overview";
  }

  function showRoute(moveFocus) {
    const name = currentRoute();
    ROUTES.forEach(function (route) {
      byId("view-" + route).hidden = route !== name;
    });
    document.querySelectorAll("nav a").forEach(function (link) {
      if (link.getAttribute("data-route") === name) {
        link.setAttribute("aria-current", "page");
      } else {
        link.removeAttribute("aria-current");
      }
    });
    document.title = TITLES[name] + " - TrackitUP";
    hideNotice();
    loaders[name]();
    if (moveFocus) {
      byId("title-" + name).focus();
    }
  }

  window.addEventListener("hashchange", function () {
    showRoute(true);
  });
  showRoute(false);
})();
File 5: frontend/css/styles.css
css
/* TrackitUP - passbook style: blue, white and green */

:root {
  --ink: #14263b;
  --muted: #566a80;
  --page: #eaf1f8;
  --paper: #ffffff;
  --blue-900: #0e2745;
  --blue-700: #1b4f8f;
  --blue-100: #dce9f6;
  --blue-50: #f2f7fc;
  --green-700: #16754d;
  --green-500: #2e9e6b;
  --green-100: #ddf2e6;
  --alert: #a3402a;
  --alert-bg: #fbeae4;
  --line: #d3dfec;
  --slab: "Zilla Slab", Georgia, "Times New Roman", serif;
  --sans: "Public Sans", system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
}

*, *::before, *::after { box-sizing: border-box; }

html { font-size: 100%; }

body {
  margin: 0;
  background: var(--page);
  color: var(--ink);
  font-family: var(--sans);
  font-size: 1rem;
  line-height: 1.5;
}

h1, h2, h3, p, dl, dd, dt, ul, ol { margin: 0; padding: 0; }
ul, ol { list-style: none; }

a { color: var(--blue-700); }
a:hover { color: var(--blue-900); }

:focus-visible {
  outline: 3px solid #5b9bde;
  outline-offset: 2px;
}

.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}

.skip-link {
  position: absolute;
  left: 12px;
  top: -60px;
  background: var(--paper);
  color: var(--blue-900);
  padding: 8px 14px;
  border-radius: 4px;
  z-index: 10;
}
.skip-link:focus { top: 12px; }

.muted { color: var(--muted); }

/* ---------- top bar ---------- */

.topbar {
  background: var(--blue-900);
  color: #fff;
}

.topbar-inner {
  max-width: 880px;
  margin: 0 auto;
  padding: 0 16px;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 4px 24px;
}

.brand {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 1.6rem;
  padding: 14px 0 12px;
  letter-spacing: 0.01em;
}

.topbar nav { display: flex; flex-wrap: wrap; }

.topbar nav a {
  color: #d3e2f4;
  text-decoration: none;
  padding: 12px 14px 10px;
  border-bottom: 3px solid transparent;
  font-weight: 500;
}
.topbar nav a:hover { color: #fff; }
.topbar nav a[aria-current="page"] {
  color: #fff;
  border-bottom-color: var(--green-500);
}

/* ---------- page sheet (the passbook page) ---------- */

main {
  max-width: 880px;
  margin: 0 auto;
  padding: 24px 16px 56px;
}

.sheet {
  background: var(--paper);
  border: 1px solid var(--line);
  border-radius: 6px;
  padding: 28px 28px 32px 76px;
  background-image: linear-gradient(
    to right,
    transparent 46px, var(--blue-100) 46px, var(--blue-100) 47px, transparent 47px,
    transparent 51px, var(--blue-100) 51px, var(--blue-100) 52px, transparent 52px
  );
}
.sheet[hidden] { display: none; }

h1 {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 2rem;
  line-height: 1.2;
  color: var(--blue-900);
  margin-bottom: 20px;
}
h1:focus { outline: none; }

h2 {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 1.3rem;
  color: var(--blue-900);
  margin-bottom: 10px;
}

h3 {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 1.1rem;
  margin: 18px 0 8px;
}

.intro { color: var(--muted); max-width: 60ch; margin: -8px 0 20px; }

.small-label { color: var(--muted); font-weight: 500; font-size: 0.9rem; }

.category { text-transform: capitalize; font-weight: 500; }

/* ---------- notice bar ---------- */

.notice {
  margin-bottom: 14px;
  padding: 12px 16px;
  border-radius: 4px;
  border-left: 4px solid;
  white-space: pre-line;
}
.notice[hidden] { display: none; }
.notice-error { background: var(--alert-bg); border-color: var(--alert); color: #6d2615; }
.notice-success { background: var(--green-100); border-color: var(--green-700); color: #0d4a30; }

/* ---------- overview ---------- */

.balance-block { margin-bottom: 22px; }

.balance-figure {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 3.6rem;
  line-height: 1.1;
  color: var(--blue-900);
  font-variant-numeric: tabular-nums;
}
.balance-note { color: var(--muted); margin-top: 2px; }
.is-negative .balance-figure { color: var(--alert); }
.is-negative .balance-note { color: var(--alert); }

.split { margin-bottom: 32px; }

.split-bar {
  display: flex;
  height: 16px;
  background: var(--blue-50);
  border: 1px solid var(--line);
  border-radius: 3px;
  overflow: hidden;
  gap: 2px;
}
.split-bar span {
  display: block;
  width: 0;
  height: 100%;
  transition: width 0.9s cubic-bezier(0.2, 0.7, 0.2, 1);
}
.split-income { background: var(--green-500); }
.split-expense { background: var(--blue-700); }

.split-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 36px;
  margin-top: 12px;
}
.split-legend dt { display: flex; align-items: center; gap: 8px; color: var(--muted); font-size: 0.9rem; }
.split-legend dd { font-weight: 600; font-size: 1.15rem; font-variant-numeric: tabular-nums; }

.swatch { width: 12px; height: 12px; border-radius: 2px; display: inline-block; }
.swatch-income { background: var(--green-500); }
.swatch-expense { background: var(--blue-700); }

.two-col {
  display: grid;
  grid-template-columns: 1fr 1.4fr;
  gap: 32px;
  align-items: start;
}

.largest-amount {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 2.2rem;
  line-height: 1.1;
  color: var(--blue-700);
  font-variant-numeric: tabular-nums;
}
.largest-category { text-transform: capitalize; font-weight: 600; margin-top: 4px; }

.ledger-row {
  display: grid;
  grid-template-columns: 6.5rem 1fr auto;
  gap: 12px;
  padding: 9px 0;
  border-bottom: 1px solid var(--line);
  align-items: baseline;
}
.ledger-date { color: var(--muted); font-size: 0.9rem; font-variant-numeric: tabular-nums; }
.ledger-main { display: flex; flex-direction: column; min-width: 0; }
.note-line { font-size: 0.85rem; overflow-wrap: anywhere; }

.amount { font-weight: 600; font-variant-numeric: tabular-nums; white-space: nowrap; }
.amount-income { color: var(--green-700); }
.amount-expense { color: var(--blue-700); }
.amount-negative { color: var(--alert); }

.more-link { margin-top: 12px; }

/* ---------- transactions table ---------- */

.empty { padding: 12px 0; color: var(--muted); }
.empty[hidden] { display: none; }

.ledger-table { width: 100%; border-collapse: collapse; }
.ledger-table[hidden] { display: none; }

.ledger-table th {
  text-align: left;
  font-weight: 600;
  font-size: 0.9rem;
  color: var(--muted);
  padding: 8px 8px 8px 0;
  border-bottom: 2px solid var(--blue-700);
}
.ledger-table td {
  padding: 10px 8px 10px 0;
  border-bottom: 1px solid var(--line);
  vertical-align: baseline;
}
.ledger-table .num { text-align: right; }
.col-number { color: var(--muted); font-variant-numeric: tabular-nums; }
.col-date { font-variant-numeric: tabular-nums; white-space: nowrap; }
.col-category { text-transform: capitalize; font-weight: 500; }
.col-note { color: var(--muted); overflow-wrap: anywhere; }
.col-action { text-align: right; white-space: nowrap; padding-right: 0; }

.tag {
  display: inline-block;
  padding: 1px 8px;
  border-radius: 3px;
  font-size: 0.85rem;
  font-weight: 500;
  border: 1px solid;
}
.tag-income { color: var(--green-700); border-color: var(--green-500); background: var(--green-100); }
.tag-expense { color: var(--blue-700); border-color: #9bb9dc; background: var(--blue-50); }

.btn-delete {
  font: inherit;
  font-size: 0.9rem;
  color: var(--alert);
  background: transparent;
  border: 1px solid var(--alert);
  border-radius: 4px;
  padding: 4px 10px;
  cursor: pointer;
}
.btn-delete:hover { background: var(--alert-bg); }
.btn-delete.armed { background: var(--alert); color: #fff; }
.btn-delete:disabled { opacity: 0.6; cursor: default; }

/* ---------- forms ---------- */

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px 20px;
  max-width: 560px;
}
.field { display: flex; flex-direction: column; gap: 6px; }
.field-wide { grid-column: 1 / -1; }

label { font-weight: 500; font-size: 0.95rem; }

input, select {
  font: inherit;
  color: var(--ink);
  background: #fff;
  height: 44px;
  padding: 0 12px;
  border: 1px solid #94aac1;
  border-radius: 4px;
  width: 100%;
}
input:focus, select:focus { border-color: var(--blue-700); }

.btn {
  font: inherit;
  font-weight: 600;
  color: #fff;
  border: 0;
  border-radius: 4px;
  height: 44px;
  padding: 0 22px;
  cursor: pointer;
}
.btn-green { background: var(--green-700); }
.btn-green:hover { background: #0f5f3d; }
.btn-blue { background: var(--blue-700); }
.btn-blue:hover { background: var(--blue-900); }
.btn:disabled { opacity: 0.6; cursor: default; }

.form-result {
  margin-top: 18px;
  max-width: 560px;
  padding: 12px 16px;
  border-radius: 4px;
  border-left: 4px solid;
  white-space: pre-line;
}
.form-result[hidden] { display: none; }
.form-result-success { background: var(--green-100); border-color: var(--green-700); color: #0d4a30; }
.form-result-error { background: var(--alert-bg); border-color: var(--alert); color: #6d2615; }

/* ---------- reports ---------- */

.cat-row { margin-bottom: 14px; }
.cat-top { display: flex; justify-content: space-between; gap: 12px; margin-bottom: 4px; }
.cat-track { height: 8px; background: var(--blue-50); border-radius: 2px; }
.cat-fill { height: 100%; border-radius: 2px; }
.cat-fill-expense { background: var(--blue-700); }
.cat-fill-income { background: var(--green-500); }

.month-block { margin-top: 36px; padding-top: 24px; border-top: 2px solid var(--blue-100); }

.month-form { display: flex; align-items: flex-end; gap: 14px; flex-wrap: wrap; }
.month-form .field { width: 12rem; }

.field-error { color: var(--alert); margin-top: 10px; }
.field-error[hidden] { display: none; }

.month-figures { display: flex; flex-wrap: wrap; gap: 12px 40px; }
.month-figure dt { color: var(--muted); font-size: 0.9rem; }
.month-figure dd {
  font-family: var(--slab);
  font-weight: 600;
  font-size: 1.6rem;
  font-variant-numeric: tabular-nums;
}

/* ---------- small screens ---------- */

@media (max-width: 720px) {
  .sheet { padding: 22px 18px 26px; background-image: none; }
  .two-col { grid-template-columns: 1fr; gap: 28px; }
  .balance-figure { font-size: 2.8rem; }
  .form-grid { grid-template-columns: 1fr; }

  .ledger-table thead { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); }
  .ledger-table, .ledger-table tbody, .ledger-table tr, .ledger-table td { display: block; width: 100%; }
  .ledger-table tr { padding: 10px 0; border-bottom: 1px solid var(--line); }
  .ledger-table td { border: 0; padding: 2px 0; display: flex; justify-content: space-between; gap: 16px; }
  .ledger-table td::before { content: attr(data-label); color: var(--muted); font-size: 0.9rem; font-weight: 400; }
  .ledger-table td.col-action { justify-content: flex-end; padding-top: 6px; }
  .ledger-table td.col-action::before { content: none; }
}

@media (prefers-reduced-motion: reduce) {
  .split-bar span { transition: none; }
}
How each screen connects to your backend
Screen / action	Web address (in server.py)	Your function that runs
Overview: balance, income, expenses	GET /api/summary	transaction_service.summary()
Overview: largest expense	GET /api/reports/largest-expense	largest_expense()
Overview and Transactions: list	GET /api/transactions	get_all_transactions()
Add form	POST /api/transactions	add_transaction(date, type_, category, amount, note)
Delete button	DELETE /api/transactions/{number}	delete_transaction(index)
Reports: category bars	GET /api/reports/categories	category_report()
Reports: month summary	GET /api/reports/monthly?month=2026-10	monthly_report(month)

Every error message you see in the UI is the exact text your validator.py and transaction_service.py return. This includes “Date cannot be in the future.” and the negative-balance WARNING after an add or delete.

Things I noticed in your backend (not changed)
Delete uses the row’s position, not an ID. After a delete, every number below it shifts up by one. The UI re-checks the row against a fresh copy of the list just before deleting. If the file changed, it deletes nothing and reloads.
Broken CSV rows disappear silently. storage.py skips them. If you edit the file in Excel and make a mistake, that row vanishes from the app. It is still in the file until the next save, which then rewrites the file without it.
Don’t run main.py and server.py together. Both rewrite the same CSV file, and each only sees the other’s changes when it next reads the file.
No currency symbol is shown, because your code doesn’t define one. If you want ₹, change formatMoney in app.js.
Category names show capitalised (“Food”). Your validator stores them in lowercase and I left that as is.
A 1 AM test caught a date edge. The Add form fills in today’s date from the browser clock, and your validator checks it against Python’s clock. On the same laptop these always agree. They differed only on my test machine, where the two clocks sat on different calendar days.

If you hit any error while running this, paste the terminal output and I’ll trace it with you.




Claude is AI and can make mistakes.
Git workflow · MD
Git workflow, push order and test steps

Five members work at the same time. Nobody waits for anybody to write code. The merge order below decides when each piece goes into main, so every merge is clean and every step is tested.

1. Rules
Nobody pushes straight to main. Everything goes through a Pull Request (PR) with 1 approval.
One branch per module. One PR contains only your module file and your test file.
Nobody edits another member's file. If you need a change there, ask the owner in the group chat.
constants.py and transaction.py are shared. Change them only in a separate PR that all five members review.
Before you open a PR: your own test passes, and python run_all_tests.py has no FAIL.
Merge with "Create a merge commit" so everyone's small commits stay visible in the history.
Never commit transactions.csv. It is in .gitignore.
2. Who owns what
Member	Module	Branch	Files in your PR	Reviewer
Repo owner	starter commit	(direct to main, once)	everything in the starter folder	all
3	Storage	feature/storage	storage.py, test_storage.py	Member 2
2	Validation	feature/validation	validator.py, test_validator.py	Member 3
5	Reports	feature/reports	reports.py, test_reports.py	Member 4
4	Service	feature/service	transaction_service.py, test_service.py, test_integration.py	Member 1
1	Menu	feature/menu	menu.py, test_menu.py, test_end_to_end.py	Member 4
3. Merge order
Step	What goes into main	Why now	Gate before the merge button
0	Starter commit (repo owner)	Gives everyone the shared files, stubs, docs and sample data	python main.py runs and prints the stub message
1	Storage (Member 3)	The data layer everything else sits on	python test_storage.py passes
2	Validation (Member 2)	Needs only constants. Independent of step 1	python test_validator.py passes
3	Reports (Member 5)	Needs only Transaction. Independent of steps 1 and 2	python test_reports.py passes
4	Service (Member 4)	Needs the real steps 1, 2 and 3. Now the integration test can run	python test_service.py and python test_integration.py pass
5	Menu (Member 1)	Needs the real service. Now the whole app can run	python test_menu.py, python test_end_to_end.py and python run_all_tests.py pass
6	Release v1.0	Everything is merged and tested	demo run from section 7 done

Steps 1 to 3 are independent, so their PRs can be reviewed at the same time. Keep the order anyway, so the history reads clearly. Steps 4 and 5 must wait for the steps before them.

Why this order works: every file is touched by exactly one PR, so Git never sees two people changing the same lines. The starter commit already contains a stub for each module, so each PR simply replaces a stub with the real file.

4. Step by step
Step 0 - Repo owner (once)
powershell
# copy everything from the 1_starter_commit folder into your TrackitUP folder, then:
cd TrackitUP
python run_all_tests.py        # test_transaction passes, the others say SKIP
python main.py                 # prints: Menu is not built yet. (stub)
git add .
git commit -m "chore: starter commit with shared files, stubs, docs and sample data"
git push origin main

Then on GitHub:

Settings -> Branches (or Rules -> Rulesets) -> add a rule for main: require a pull request, require 1 approval, and tick "Require branches to be up to date before merging". Turn this on after the push above.
Create 5 Issues, one per module, and assign each member.
Everyone: GitHub Desktop -> Fetch origin -> Pull origin.
Step 1 - Everyone, at the same time

Terminal version (GitHub Desktop version in brackets):

powershell
git checkout main
git pull                                     # [Fetch origin, then Pull origin]
git checkout -b feature/storage              # [Current Branch -> New Branch]  use YOUR branch name
Open your file in VS Code. It is a stub right now. Replace the whole file with your real module code. Create your test file(s) from the list in section 2.
Run your test (section 5). Fix until it passes.
Commit small and often:
powershell
git add storage.py test_storage.py
git commit -m "feat: load and save transactions in CSV"
git push -u origin feature/storage           # [Publish branch / Push origin]
Open a Pull Request (GitHub Desktop: Branch -> Create Pull Request). Fill in the template and write Closes #<your issue number>.

Commit message style:

Start with	Use for	Example
feat:	new behaviour	feat: reject future dates
fix:	bug fix	fix: skip rows with a wrong date
test:	tests only	test: add broken row cases
docs:	documents and comments	docs: explain the CSV columns
refactor:	cleaner code, same behaviour	refactor: split row conversion
Step 2 - Merge, one after the other

Repeat this for steps 1, 2, 3, 4, 5 in order:

The reviewer opens the PR, reads the code, checks the template boxes, and approves or comments.
The author fixes comments and pushes again.
When approved and the gate in section 3 is green, click Merge pull request -> Create a merge commit, then Delete branch.
Everyone else brings the new main into their own branch:
powershell
git checkout feature/service                 # your own branch
git fetch origin
git merge origin/main                        # [Branch -> Update from main]
python run_all_tests.py

If the PR page says "This branch is out-of-date", click Update branch on GitHub. This is the normal sequential flow, not an error.

Step 3 - Release

After step 5:

powershell
git checkout main
git pull
python run_all_tests.py                      # everything PASS, nothing SKIP

Do the demo in section 7. Then on GitHub: Releases -> Draft a new release -> tag v1.0 -> target main -> Publish.

5. Tests: what each member runs
Member	Module test (runs alone)	Needs other modules?	Later test
3 Storage	python test_storage.py	no	
2 Validation	python test_validator.py	no	
5 Reports	python test_reports.py	no	
4 Service	python test_service.py (uses fake helpers)	no	python test_integration.py after steps 1, 2, 3 are merged
1 Menu	python test_menu.py (uses a fake service)	no	python test_end_to_end.py after step 4 is merged
Everyone	python run_all_tests.py	skips files that are not merged yet	

The three test levels:

Level	What it proves	Files
Unit (module alone)	one class does its own job	validator, storage, reports, service (with fakes), menu (with a fake service)
Integration	the service works with the real validator, storage and reports and a real CSV	test_integration.py
End to end	the real main.py works like a user typed it, including a restart	test_end_to_end.py

Test files use their own temporary CSV names that end in _tmp.csv, so your real transactions.csv is never touched.

6. Per-member checklist
Member	Code goes in	Run before the PR	PR is ready when	After your PR is merged
3	storage.py	python test_storage.py	test passes, you can explain load_transactions and row_to_transaction	help Member 4 with any storage question
2	validator.py	python test_validator.py	test passes, you tried letters, empty text and a wrong date by hand	
5	reports.py	python test_reports.py	test passes, no print() or input() in the file	
4	transaction_service.py	python test_service.py, later python test_integration.py	both pass after updating from main	tell Member 1 the service is merged
1	menu.py	python test_menu.py, later python test_end_to_end.py and python run_all_tests.py	everything passes, the demo in section 7 works	create the release
7. Demo run for the Definition of Done
powershell
python main.py
Option 1, add five transactions (a salary, then four expenses in different categories). Try one wrong date and one abc amount to show the error messages.
Option 2 shows the list with numbers.
Option 3 deletes one by its number.
Options 4, 5, 6 (type 2026-10) and 7 show the reports.
Add an expense bigger than your balance to show the negative balance warning.
Option 8 exits. Run python main.py again and choose 2. The records are still there.
Open transactions.csv in Excel (close Excel afterwards).

Quick demo with ready-made data:

powershell
Copy-Item sample_transactions.csv transactions.csv
python main.py
8. If something goes wrong
Problem	Fix
Push rejected	Someone merged first. Run git fetch origin, then git merge origin/main, then push again
Merge conflict in a file you did not edit	You edited someone else's file by mistake. git checkout origin/main -- <that file>
Merge conflict in your own file	Should not happen. Tell the group chat before fixing it
You committed transactions.csv	git rm --cached transactions.csv, commit, push. Check .gitignore
You are on main and have edits	git checkout -b feature/<your module> keeps your edits on a new branch
Test says SKIP	The file is not merged yet. That is normal before its step
9. What this history shows your teacher
Issues with an owner for each module
One branch and one PR per member, each linked to its issue
Reviews and approvals from a different member
Merge commits in dependency order
Tests at three levels and a tagged release v1.0
