# johnchen.github.io
import tkinter as tk
from tkinter import ttk, messagebox
import requests
from bs4 import BeautifulSoup
from database import PaperDatabase  # 导入数据库类


class ArxivCrawlerGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("arXiv论文监控器 v0.9")
        self.root.geometry("1000x700")

        # 初始化数据库
        self.db = PaperDatabase()

        self.论文数据 = []
        self.创建界面()

    def 创建界面(self):
        """创建完整的GUI界面"""
        # 控制面板
        控制框架 = ttk.LabelFrame(self.root, text="搜索设置", padding="10")
        控制框架.pack(fill=tk.X, padx=10, pady=5)

        ttk.Label(控制框架, text="分类:").grid(row=0, column=0, sticky=tk.W, padx=(0, 5))
        self.分类输入 = ttk.Entry(控制框架, width=30)
        self.分类输入.grid(row=0, column=1, padx=(0, 10))
        self.分类输入.insert(0, "cond-mat.supr-con")

        ttk.Label(控制框架, text="数量:").grid(row=0, column=2, sticky=tk.W, padx=(0, 5))
        self.数量输入 = ttk.Entry(控制框架, width=10)
        self.数量输入.grid(row=0, column=3, padx=(0, 10))
        self.数量输入.insert(0, "5")

        self.开始按钮 = ttk.Button(控制框架, text="开始爬取", command=self.开始爬取)
        self.开始按钮.grid(row=0, column=4, padx=10)

        # 添加历史记录按钮
        self.历史按钮 = ttk.Button(控制框架, text="查看历史", command=self.显示历史记录)
        self.历史按钮.grid(row=0, column=5, padx=10)

        # 状态显示
        self.状态标签 = ttk.Label(self.root, text="准备就绪", padding="5")
        self.状态标签.pack(fill=tk.X, padx=10)

        # 论文表格显示区域
        self.创建论文表格()

    def 创建论文表格(self):
        """创建论文列表表格"""
        表格框架 = ttk.LabelFrame(self.root, text="论文列表", padding="10")
        表格框架.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        列 = ('标题', '作者', '链接')
        self.论文表格 = ttk.Treeview(表格框架, columns=列, show='headings', height=15)

        self.论文表格.heading('标题', text='论文标题')
        self.论文表格.heading('作者', text='作者')
        self.论文表格.heading('链接', text='链接')

        self.论文表格.column('标题', width=400, anchor='w')
        self.论文表格.column('作者', width=200, anchor='w')
        self.论文表格.column('链接', width=300, anchor='w')

        滚动条 = ttk.Scrollbar(表格框架, orient=tk.VERTICAL, command=self.论文表格.yview)
        self.论文表格.configure(yscrollcommand=滚动条.set)

        self.论文表格.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        滚动条.pack(side=tk.RIGHT, fill=tk.Y)

        self.论文表格.bind('<Double-1>', self.打开论文链接)

    def 打开论文链接(self, event):
        选中项 = self.论文表格.selection()
        if 选中项:
            项数据 = self.论文表格.item(选中项[0])
            链接 = 项数据['values'][2]
            import webbrowser
            webbrowser.open(链接)

    def 开始爬取(self):
        try:
            分类 = self.分类输入.get().strip()
            数量文本 = self.数量输入.get().strip()

            if not 分类:
                messagebox.showerror("输入错误", "请输入分类名称")
                return

            if not 数量文本.isdigit():
                messagebox.showerror("输入错误", "请输入有效的数字")
                return

            数量 = int(数量文本)
            if 数量 <= 0 or 数量 > 50:
                messagebox.showerror("输入错误", "数量应在1-50之间")
                return

            self.状态标签.config(text="正在爬取...")
            self.开始按钮.config(state=tk.DISABLED)

            # 执行爬取
            self.执行爬取任务(分类, 数量)

        except ValueError as e:
            messagebox.showerror("错误", f"数据格式错误: {str(e)}")
        except Exception as e:
            messagebox.showerror("错误", f"爬取失败: {str(e)}")
        finally:
            self.开始按钮.config(state=tk.NORMAL)

    def 执行爬取任务(self, 分类, 数量):
        try:
            url = f'https://arxiv.org/list/{分类}/new'
            response = requests.get(url, timeout=10)
            response.raise_for_status()

            soup = BeautifulSoup(response.text, 'html.parser')
            dl_list = soup.find('dl')

            if not dl_list:
                messagebox.showinfo("提示", "未找到论文数据")
                return

            dt_list = dl_list.find_all('dt')[:数量]
            dd_list = dl_list.find_all('dd')[:数量]

            # 清空表格
            for item in self.论文表格.get_children():
                self.论文表格.delete(item)

            # 填充新数据
            for i in range(len(dt_list)):
                paper_info = self.提取论文信息(dt_list[i], dd_list[i])
                if paper_info:
                    self.论文表格.insert('', 'end', values=(
                        paper_info.get('标题', ''),
                        paper_info.get('作者', ''),
                        paper_info.get('链接', '')
                    ))
                    # 保存到数据库
                    self.db.insert_paper(paper_info, 分类)

            self.状态标签.config(text=f"成功获取 {len(dt_list)} 篇论文")

        except requests.exceptions.RequestException as e:
            messagebox.showerror("网络错误", f"网络请求失败: {str(e)}")
        except Exception as e:
            messagebox.showerror("解析错误", f"数据解析失败: {str(e)}")

    def 提取论文信息(self, dt标签, dd标签):
        paper_info = {}

        link_tag = dt标签.find('a', title='Abstract')
        if link_tag:
            paper_info['链接'] = "https://arxiv.org" + link_tag['href']

        title_div = dd标签.find('div', class_='list-title')
        if title_div:
            paper_info['标题'] = title_div.text.replace('Title:', '').strip()[:100]

        authors_div = dd标签.find('div', class_='list-authors')
        if authors_div:
            paper_info['作者'] = authors_div.text.replace('Authors:', '').strip()[:80]

        abstract_p = dd标签.find('p', class_='mathjax')
        if abstract_p:
            paper_info['摘要'] = abstract_p.text.strip()

        return paper_info

    def 显示历史记录(self):
        # 创建新窗口显示历史记录
        历史窗口 = tk.Toplevel(self.root)
        历史窗口.title("历史爬取记录")
        历史窗口.geometry("800x600")

        # 创建表格
        列 = ('id', '标题', '作者', '分类', '时间')
        历史表格 = ttk.Treeview(历史窗口, columns=列, show='headings', height=20)

        历史表格.heading('id', text='ID')
        历史表格.heading('标题', text='标题')
        历史表格.heading('作者', text='作者')
        历史表格.heading('分类', text='分类')
        历史表格.heading('时间', text='爬取时间')

        历史表格.column('id', width=40)
        历史表格.column('标题', width=300)
        历史表格.column('作者', width=150)
        历史表格.column('分类', width=100)
        历史表格.column('时间', width=120)

        滚动条 = ttk.Scrollbar(历史窗口, orient=tk.VERTICAL, command=历史表格.yview)
        历史表格.configure(yscrollcommand=滚动条.set)

        历史表格.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        滚动条.pack(side=tk.RIGHT, fill=tk.Y)

        # 从数据库获取数据
        papers = self.db.get_all_papers()
        for paper in papers:
            历史表格.insert('', 'end', values=paper)


# 启动应用
if __name__ == "__main__":
    root = tk.Tk()
    app = ArxivCrawlerGUI(root)
    root.mainloop()
