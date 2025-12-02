# My-First-Python-Project
#!/usr/bin/env python3
"""
student_manager_compact.py
Compact Student Manager: CSV + GUI + Reports + Export (Maths, Chemistry, Physics)
"""

import csv, os, webbrowser, tkinter as tk
from tkinter import ttk, messagebox, simpledialog

CSV="report_cards.csv"
SUBS=["Maths","Chemistry","Physics"]
FIELDS=["roll","name"]+SUBS+["total","percent","grade","result"]
PASS=33

# ---- data helpers ----
def ensure():
    if not os.path.exists(CSV):
        with open(CSV,"w",newline="") as f: csv.writer(f).writerow(FIELDS)

def read_all():
    out=[]
    if os.path.exists(CSV):
        with open(CSV,newline="") as f:
            for r in csv.DictReader(f): out.append(r)
    return out

def write_all(rows):
    with open(CSV,"w",newline="") as f:
        w=csv.DictWriter(f,fieldnames=FIELDS); w.writeheader(); w.writerows(rows)

def compute(marks):
    total=sum(marks); pct=round(total/len(marks),2)
    g = "A+" if pct>=90 else "A" if pct>=80 else "B" if pct>=70 else "C" if pct>=60 else "D"
    res = "Pass" if all(m>=PASS for m in marks) else "Fail"
    return str(total), str(pct), g, res

# ---- stats & export ----
def class_stats(rows):
    if not rows: return None
    pts=[float(r["percent"]) for r in rows]
    passed=sum(1 for r in rows if r["result"]=="Pass")
    return {"count":len(pts),"avg":round(sum(pts)/len(pts),2),"high":max(pts),"low":min(pts),"passed":passed,"failed":len(pts)-passed}

def subject_stats(rows):
    s={}
    for sub in SUBS:
        vals=[int(r[sub]) for r in rows] if rows else []
        s[sub]={"avg":round(sum(vals)/len(vals),2) if vals else None,"high":max(vals) if vals else None,"low":min(vals) if vals else None,"pass":sum(1 for v in vals if v>=PASS)}
    return s

def export_html(rows):
    path=os.path.abspath("report_cards_report.html")
    html=["<html><meta charset='utf-8'><title>Report</title><style>body{font-family:Arial}table{border-collapse:collapse;width:100%}th,td{border:1px solid #ccc;padding:6px;text-align:center}th{background:#333;color:#fff}</style><body>"]
    c=class_stats(rows)
    html.append("<h2>Class Report</h2>")
    if not c: html.append("<p>No records</p>")
    else:
        html.append(f"<p>Students: {c['count']} &nbsp; Avg%: {c['avg']} &nbsp; High: {c['high']} &nbsp; Low: {c['low']} &nbsp; Passed: {c['passed']}</p>")
        html.append("<h3>Subject-wise</h3><table><tr><th>Subject</th><th>Avg</th><th>High</th><th>Low</th><th>Pass</th></tr>")
        for sub,v in subject_stats(rows).items():
            html.append(f"<tr><td>{sub}</td><td>{v['avg'] if v['avg'] is not None else '-'}</td><td>{v['high'] if v['high'] is not None else '-'}</td><td>{v['low'] if v['low'] is not None else '-'}</td><td>{v['pass']}</td></tr>")
        html.append("</table><h3>All Students</h3><table><tr>"+ "".join(f"<th>{h}</th>" for h in FIELDS) +"</tr>")
        for r in rows: html.append("<tr>"+ "".join(f"<td>{r[h]}</td>" for h in FIELDS) +"</tr>")
        html.append("</table>")
    html.append("</body></html>")
    with open(path,"w",encoding="utf-8") as f: f.write("\n".join(html))
    webbrowser.open("file://"+path)

# ---- GUI ----
class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Student Manager (Compact)"); self.geometry("980x540")
        ensure(); self._build(); self.load_table()

    def _build(self):
        top=ttk.Frame(self,padding=8); top.pack(fill="x")
        ttk.Label(top,text="Roll").grid(row=0,column=0,sticky="w"); self.e_roll=ttk.Entry(top,width=12); self.e_roll.grid(row=0,column=1,padx=6)
        ttk.Label(top,text="Name").grid(row=0,column=2,sticky="w"); self.e_name=ttk.Entry(top,width=30); self.e_name.grid(row=0,column=3,padx=6)
        for i,s in enumerate(SUBS):
            ttk.Label(top,text=s).grid(row=1,column=i,pady=(8,0)); setattr(self,f"e_{s}",ttk.Entry(top,width=12)); getattr(self,f"e_{s}").grid(row=2,column=i,padx=6)
        btns=ttk.Frame(self,padding=6); btns.pack(fill="x")
        ttk.Button(btns,text="Add",command=self.add).pack(side="left",padx=6)
        ttk.Button(btns,text="Update",command=self.update).pack(side="left",padx=6)
        ttk.Button(btns,text="Delete",command=self.delete).pack(side="left",padx=6)
        ttk.Button(btns,text="Search",command=self.search).pack(side="left",padx=6)
        ttk.Button(btns,text="Clear",command=self.clear_inputs).pack(side="left",padx=6)
        ttk.Button(btns,text="Individual",command=self.indiv_report).pack(side="right",padx=6)
        ttk.Button(btns,text="Subject-wise",command=self.subj_report).pack(side="right",padx=6)
        ttk.Button(btns,text="Class Stats",command=self.class_report).pack(side="right",padx=6)
        ttk.Button(btns,text="Export HTML",command=lambda: export_html(read_all())).pack(side="right",padx=6)

        cols=FIELDS
        self.tree=ttk.Treeview(self,columns=cols,show="headings",height=18)
        for c in cols: self.tree.heading(c,text=c.capitalize()); self.tree.column(c,width=100,anchor="center")
        self.tree.column("name",width=200,anchor="w"); self.tree.pack(fill="both",expand=True,padx=8,pady=(6,8))
        self.tree.bind("<<TreeviewSelect>>",self.on_select)
        self.status=ttk.Label(self,text="Ready"); self.status.pack(fill="x")

    def load_table(self,rows=None):
        rows = read_all() if rows is None else rows
        for i in self.tree.get_children(): self.tree.delete(i)
        for r in rows: self.tree.insert("",tk.END,values=[r[h] for h in FIELDS])
        self.status.config(text=f"Loaded {len(rows)} records")

    def get_inputs(self):
        roll=self.e_roll.get().strip(); name=self.e_name.get().strip()
        try: marks=[int(getattr(self,f"e_{s}").get().strip()) for s in SUBS]
        except: messagebox.showerror("Error","Enter integer marks for all subjects"); return None
        if not roll or not name: messagebox.showerror("Error","Roll & Name required"); return None
        for m in marks:
            if m<0 or m>100: messagebox.showerror("Error","Marks must be 0-100"); return None
        return roll,name,marks

    def add(self):
        data=self.get_inputs();
        if not data: return
        roll,name,marks=data; rows=read_all()
        if any(r["roll"]==roll for r in rows): messagebox.showerror("Error","Roll exists"); return
        t,p,g,res=compute(marks); new={"roll":roll,"name":name}
        for s,m in zip(SUBS,marks): new[s]=str(m)
        new.update({"total":t,"percent":p,"grade":g,"result":res}); rows.append(new); write_all(rows); self.load_table(); self.clear_inputs(); self.status.config(text=f"Added {roll}")

    def update(self):
        sel=self.tree.selection()
        if not sel: messagebox.showinfo("Info","Select row to update"); return
        data=self.get_inputs();
        if not data: return
        roll,name,marks=data; rows=read_all(); updated=False
        for i,r in enumerate(rows):
            if r["roll"]==roll:
                t,p,g,res=compute(marks); r["name"]=name
                for s,m in zip(SUBS,marks): r[s]=str(m)
                r.update({"total":t,"percent":p,"grade":g,"result":res}); rows[i]=r; updated=True; break
        if not updated: messagebox.showerror("Error","Roll not found"); return
        write_all(rows); self.load_table(); self.clear_inputs(); self.status.config(text=f"Updated {roll}")

    def delete(self):
        sel=self.tree.selection()
        if not sel: messagebox.showinfo("Info","Select row to delete"); return
        roll=self.tree.item(sel[0],"values")[0]
        if not messagebox.askyesno("Confirm",f"Delete {roll}?"): return
        rows=[r for r in read_all() if r["roll"]!=roll]; write_all(rows); self.load_table(); self.clear_inputs(); self.status.config(text=f"Deleted {roll}")

    def search(self):
        key=simpledialog.askstring("Search","Enter roll to search")
        if not key: return
        rows=[r for r in read_all() if r["roll"]==key]
        if not rows: messagebox.showinfo("Search",f"No record for {key}"); return
        self.load_table(rows); self.status.config(text=f"Search results for {key}")

    def on_select(self,_e):
        sel=self.tree.selection()
        if not sel: return
        vals=self.tree.item(sel[0],"values")
        self.e_roll.delete(0,"end"); self.e_roll.insert(0,vals[0])
        self.e_name.delete(0,"end"); self.e_name.insert(0,vals[1])
        for i,s in enumerate(SUBS): getattr(self,f"e_{s}").delete(0,"end"); getattr(self,f"e_{s}").insert(0,vals[2+i])

    def clear_inputs(self):
        self.e_roll.delete(0,"end"); self.e_name.delete(0,"end")
        for s in SUBS: getattr(self,f"e_{s}").delete(0,"end")

    # reports
    def indiv_report(self):
        sel=self.tree.selection();
        if not sel: messagebox.showinfo("Info","Select student"); return
        vals=self.tree.item(sel[0],"values"); roll=vals[0]
        rows=[r for r in read_all() if r["roll"]==roll]
        if not rows: messagebox.showerror("Error","Record not found"); return
        r=rows[0]; txt=f"Name: {r['name']}\nRoll: {r['roll']}\n"
        for s in SUBS: txt+=f"{s}: {r[s]}\n"
        txt+=f"\nTotal: {r['total']}\nPercent: {r['percent']}\nGrade: {r['grade']}\nResult: {r['result']}"
        messagebox.showinfo("Individual Report",txt)

    def subj_report(self):
        s=subject_stats(read_all()); txt="Subject-wise Stats\n\n"
        for sub,v in s.items(): txt+=f"{sub} — Avg: {v['avg'] if v['avg'] is not None else '-'}  High: {v['high'] if v['high'] is not None else '-'}  Low: {v['low'] if v['low'] is not None else '-'}  Pass: {v['pass']}\n"
        messagebox.showinfo("Subject-wise Report",txt)

    def class_report(self):
        c=class_stats(read_all())
        if not c: messagebox.showinfo("Class Stats","No records"); return
        txt=f"Students: {c['count']}\nAvg%: {c['avg']}\nHigh%: {c['high']}\nLow%: {c['low']}\nPassed: {c['passed']}\nFailed: {c['failed']}"
        messagebox.showinfo("Class Statistics",txt)

if __name__=="__main__":
    App().mainloop()
