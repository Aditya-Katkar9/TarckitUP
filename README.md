# TarckitUP
 These are for **Windows PowerShell**, since your laptops run Windows. I read "Step 0" as one-time setup on every laptop, and "Step 1" as Ananya creating the project structure once.

## Step 0: every team member (one time)

**1. Check the tools.** Open a terminal (in VS Code press `` Ctrl+` ``):
```powershell
python --version
git --version
```
If `git --version` says "not recognized", GitHub Desktop doesn't put Git in the terminal. Either install Git from git-scm.com, or do the commit and push parts in GitHub Desktop instead.

**2. Clone the repo.** In GitHub Desktop use **File → Clone repository → TrackitUP**, then click **Open in Visual Studio Code**. Or use the terminal:
```powershell
cd $HOME\Documents
git clone https://github.com/ananyaladda-parkar/TrackitUP.git
cd TrackitUP
code .
```

## Step 1: Ananya only (project structure)

Run these inside the `TrackitUP` folder:

```powershell
# 1. Create the empty module files
New-Item main.py, menu.py, validator.py, storage.py, transaction_service.py, reports.py, README.md, requirements.txt, .gitignore -ItemType File

# 2. .gitignore (keeps test data and the virtual environment out of Git)
@"
transactions.csv
__pycache__/
.venv/
"@ | Set-Content .gitignore -Encoding ascii

# 3. requirements.txt
@"
# No external packages needed. This project uses only Python's standard library.
"@ | Set-Content requirements.txt -Encoding ascii

# 4. Starter README
"# TrackitUP - Personal Expense Tracker" | Set-Content README.md -Encoding ascii

# 5. Commit and push the first structure
git add .
git commit -m "chore: initial project structure"
git push origin main
```

`requirements.txt` has no packages because the code only uses Python's built-in `csv`, `datetime`, `os` and `math`.

Your folder now looks like this:
```
TrackitUP/
├── main.py                    (Member 1)
├── menu.py                    (Member 1)
├── validator.py               (Member 2)
├── storage.py                 (Member 3)
├── transaction_service.py     (Member 4)
├── reports.py                 (Member 5)
├── requirements.txt
├── README.md
└── .gitignore
```

**After this push, turn on the branch protection rule** (Settings → Branches), as planned earlier. If you turn it on first, this first push gets blocked.

## Step 1b: every member (optional virtual environment)

This keeps your Python packages separate per project. It's good practice, though not required here:
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```
If PowerShell says "running scripts is disabled", run this once, then activate again:
```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```
In VS Code, if it asks to use the new environment, click **Yes**.

## Next: your own branch and code

After Ananya's push, everyone pulls (**Fetch origin → Pull origin**), then creates their feature branch:
```powershell
git checkout -b feature/storage     # use your own module name
```
Paste your module code into your file, then `git add .`, `git commit`, `git push -u origin feature/storage`, and open a Pull Request.

Want the full README (setup, how to run, module ownership table) next?
