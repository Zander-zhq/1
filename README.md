import threading
import tkinter as tk
from tkinter import filedialog, ttk, messagebox, scrolledtext
import os
import sys

from file import delete_parent_folder
from control import Data_collection_tasks, load_data
from crop import VideoCropper


class TextRedirector:
    def __init__(self, text_widget):
        self.text_widget = text_widget

    def write(self, string):
        # 由于我们可能会在非GUI线程中调用此方法，因此需要使用线程安全的方式来更新文本小部件
        def update_text_widget():
            self.text_widget.configure(state='normal')
            self.text_widget.insert('end', string)
            self.text_widget.configure(state='disabled')  # 防止用户编辑，但仍然可以滚动
            self.text_widget.see('end')  # 滚动到文本区域的末尾

        # 使用GUI线程的事件循环来执行更新
        self.text_widget.after_idle(update_text_widget)


class VideoProcessorApp:
    def __init__(self, master):
        self.video_list = None
        self.master = master
        self.master.title("视频处理应用")
        self.create_widgets()

    def create_widgets(self):
        # 设置整体布局
        self.master.geometry('900x700')  # 调整窗口大小以适应新的输入框
        self.master.resizable(False, False)

        # 创建一个容纳所有组件的主框架
        main_frame = ttk.Frame(self.master)
        main_frame.grid(row=0, column=0, padx=10, pady=10, sticky='nsew')

        # 配置主框架的列和行权重
        main_frame.columnconfigure(1, weight=1)
        main_frame.rowconfigure(4, weight=1)

        # 输出目录
        ttk.Label(main_frame, text="输出目录").grid(row=0, column=0, padx=10, pady=5, sticky='w')
        self.output_dir_entry = ttk.Entry(main_frame, width=50)
        self.output_dir_entry.grid(row=0, column=1, padx=10, pady=5, sticky='we')
        self.output_dir_entry.insert(0, "F:/aa")  # 设置默认值
        ttk.Button(main_frame, text="浏览", command=self.browse_output_dir).grid(row=0, column=2, padx=10, pady=5)
        ttk.Button(main_frame, text="查看目录", command=self.view_output_dir).grid(row=0, column=3, padx=10, pady=5)

        # 循环次数
        ttk.Label(main_frame, text="循环次数").grid(row=1, column=0, padx=10, pady=5, sticky='w')
        self.loop_count_entry = ttk.Entry(main_frame, width=50)
        self.loop_count_entry.grid(row=1, column=1, padx=10, pady=5, sticky='we')
        self.loop_count_entry.insert(0, "10")  # 设置默认值

        # CSV路径
        ttk.Label(main_frame, text="CSV路径").grid(row=2, column=0, padx=10, pady=5, sticky='w')
        self.csv_path_entry = ttk.Entry(main_frame, width=50)
        self.csv_path_entry.grid(row=2, column=1, padx=10, pady=5, sticky='we')
        self.csv_path_entry.insert(0, "F:/aa/视频数据.csv")  # 设置默认值
        ttk.Button(main_frame, text="浏览", command=self.browse_csv).grid(row=2, column=2, padx=10, pady=5)
        ttk.Button(main_frame, text="查看文件", command=self.view_csv).grid(row=2, column=3, padx=10, pady=5)

        # 创建一个单独的框架来容纳按钮
        buttons_frame = ttk.Frame(main_frame)
        buttons_frame.grid(row=3, column=0, columnspan=4, padx=10, pady=5, sticky='we')

        # 在按钮框架中配置行和列的权重（如果需要）
        buttons_frame.columnconfigure(0, weight=1)
        buttons_frame.columnconfigure(1, weight=1)
        buttons_frame.columnconfigure(2, weight=1)
        buttons_frame.columnconfigure(3, weight=1)

        # 按钮布局在按钮框架中
        ttk.Button(buttons_frame, text="数据采集", command=self.data_collection).grid(row=0, column=0, padx=5, pady=5,
                                                                                      sticky='ew')
        ttk.Button(buttons_frame, text="加载数据", command=self.load_data).grid(row=0, column=1, padx=5, pady=5,
                                                                                sticky='ew')
        ttk.Button(buttons_frame, text="清空数据", command=self.clear_treeview).grid(row=0, column=2, padx=5, pady=5,
                                                                                       sticky='ew')
        ttk.Button(buttons_frame, text="开始剪辑", command=self.start_editing).grid(row=0, column=3, padx=5, pady=5,
                                                                                    sticky='ew')

        # 视频列表
        self.video_list = ttk.Treeview(main_frame,
                                       columns=(
                                           "序号", "视频ID", "视频名称", "时长", "视频路径", "视频分辨率", "裁剪参数"),
                                       show='headings', height=10)
        # 设置每列的宽度和标题
        self.video_list.heading("序号", text="序号")
        self.video_list.column("序号", width=35, anchor='w')  # 例如，序号列宽度设置为50

        self.video_list.heading("视频ID", text="视频ID")
        self.video_list.column("视频ID", width=100, anchor='w')  # 视频ID列宽度设置为150

        self.video_list.heading("视频名称", text="视频名称")
        self.video_list.column("视频名称", width=150, anchor='w')  # 视频名称列宽度设置为150

        self.video_list.heading("时长", text="时长")
        self.video_list.column("时长", width=80, anchor='w')  # 时长列宽度设置为80

        self.video_list.heading("视频路径", text="视频路径")
        self.video_list.column("视频路径", width=150, anchor='w')  # 视频路径列宽度设置为200

        self.video_list.heading("视频分辨率", text="视频分辨率")
        self.video_list.column("视频分辨率", width=80, anchor='w')  # 视频分辨率列宽度设置为120

        self.video_list.heading("裁剪参数", text="裁剪参数")
        self.video_list.column("裁剪参数", width=80, anchor='w')  # 裁剪参数列宽度设置为120

        self.video_list.bind('<<TreeviewSelect>>', self.on_tree_select)  # 绑定选中事件

        self.video_list.grid(row=4, column=0, columnspan=4, padx=10, pady=10, sticky='nsew')

        # 滚动条
        scrollbar = ttk.Scrollbar(main_frame, orient='vertical', command=self.video_list.yview)
        scrollbar.grid(row=4, column=4, sticky='ns')
        self.video_list.configure(yscrollcommand=scrollbar.set)

        # 使用grid来放置输出文本区域
        self.output_text = scrolledtext.ScrolledText(main_frame, wrap='word', state='disabled',
                                                     font=('黑体常规', 12, 'normal'), height=10)
        self.output_text.grid(row=5, column=0, columnspan=4, padx=10, pady=10, sticky='nsew')
        # 配置主框架的最后一行权重，以便文本区域可以扩展
        main_frame.rowconfigure(5, weight=1)
        # 将标准输出重定向到文本区域
        sys.stdout = TextRedirector(self.output_text)

        # 按钮框架
        self.buttons_frame = ttk.Frame(main_frame)
        self.buttons_frame.grid(row=4, column=4, sticky='ns')

        # 初始化按钮（它们将在选中Treeview项时被更新）
        self.view_button = ttk.Button(self.buttons_frame, text="查看", command=self.view_video)
        self.view_button.pack(side='top', fill='x')
        self.delete_button = ttk.Button(self.buttons_frame, text="删除", command=self.delete_video)
        self.delete_button.pack(side='top', fill='x')
        self.crop_button = ttk.Button(self.buttons_frame, text="裁剪", command=self.crop_video)
        self.crop_button.pack(side='top', fill='x')

        # 插入测试数据
        test_data = [
            (1, 1234, "测试视频1", "00:05:23", "F:/aa/1.mp4", "1920x1080"),
            (2, 2234, "测试视频2", "00:03:15", "F:/aa/1.mp4", "1280x720"),
            (3, 5234, "测试视频3", "00:10:45", "F:/aa/1.mp4", "1920x1080"),
        ]

        # 遍历测试数据并插入到Treeview中
        for item in test_data:
            self.video_list.insert("", tk.END, iid=str(item[1]), values=item)

    # 按钮的占位符函数
    def browse_output_dir(self):
        output_dir = filedialog.askdirectory()
        if output_dir:
            self.output_dir_entry.delete(0, tk.END)
            self.output_dir_entry.insert(0, output_dir)

    def view_output_dir(self):
        output_dir = self.output_dir_entry.get()
        if os.path.exists(output_dir):
            os.startfile(output_dir)  # 使用系统默认程序打开目录（Windows系统）
        else:
            messagebox.showerror("错误", "目录不存在！")

    def browse_csv(self):
        csv_path = filedialog.askopenfilename(filetypes=[("CSV files", "*.csv")])
        if csv_path:
            self.csv_path_entry.delete(0, tk.END)
            self.csv_path_entry.insert(0, csv_path)

    def view_csv(self):
        output_dir = self.csv_path_entry.get()
        if os.path.exists(output_dir):
            os.startfile(output_dir)  # 使用系统默认程序打开目录（Windows系统）
        else:
            messagebox.showerror("错误", "目录不存在！")

    def data_collection(self):
        # 创建一个新线程来运行循环
        thread = threading.Thread(target=Data_collection_tasks,
                                  args=(self.output_dir_entry.get(), self.loop_count_entry.get(), self.video_list))
        thread.start()

    def load_data(self):
        load_data(self)

    def start_editing(self):
        # 实现开始剪辑的功能
        pass

    def clear_treeview(self):
        """清空Treeview中的所有项目"""
        for item in self.video_list.get_children():
            self.video_list.delete(item)

    def on_tree_select(self, event):
        """当Treeview中的项被选中时触发的事件"""
        self.selected_item = self.video_list.focus()  # 获取当前选中的项
        # 根据需要更新按钮的状态或执行其他操作

    def view_video(self):
        if self.selected_item:
            video_name = self.video_list.item(self.selected_item)['values'][1]
            video_path = self.video_list.item(self.selected_item)['values'][4]
            os.startfile(video_path)
            print(f"查看视频: {video_name}")
            # 这里添加查看视频的逻辑

    def delete_video(self):
        if self.selected_item:
            # 这里添加删除视频的逻辑，并确保更新self.selected_item以避免引用已删除的项
            video_path = self.video_list.item(self.selected_item)['values'][4]
            delete_parent_folder(video_path)
            self.video_list.delete(self.selected_item)
            print(f"删除视频")
            self.selected_item = None

    def crop_video(self):
        if self.selected_item:
            video_path = self.video_list.item(self.selected_item)['values'][4]
            video_ID = self.video_list.item(self.selected_item)['values'][1]
            # 创建一个新线程来运行循环
            thread = threading.Thread(target=VideoCropper,
                                      args=(video_path, video_ID, self.video_list))
            thread.start()
            print(f"裁剪视频: {video_path}")
            # 这里添加裁剪视频的逻辑
