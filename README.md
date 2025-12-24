目录概览
•	events_init.c：集中注册各个界面的 LVGL 事件回调；实现页面跳转、计算器逻辑、音乐播放逻辑、时间/闹钟设置与部分蜂鸣器控制等“业务行为”。
•	gui_guider.c：UI 总入口，初始化各 screen 的“是否已释放”标记并加载初始界面。
•	lv_dclock.c：自定义数字时钟控件（lv_dclock），并提供 24/12 小时计时函数（其中 24 小时计数使用了全局系统时间变量）。
•	music_data.c：音乐曲谱数据（频率数组、时长数组、曲目列表）。
•	setup_scr_*.c：每个界面的 UI 组件创建与样式设置（基本为自动生成），事件绑定一般调用 events_init_XXX。
•	images/*.c：图片资源（LVGL image map 数组）。
•	guider_fonts/*.c：字体资源（字模位图与 lv_font_t 定义）。
---
文件逐个说明
1) 逻辑/事件类
events_init.c
用途： 项目核心交互逻辑与事件注册中心。
主要实现：
•	统一事件入口：events_init(lv_ui *ui) 调用各页面的 events_init_XXX，完成事件绑定。
•	密码锁（Codelock）
•	监听密码输入框值变化（LV_EVENT_VALUE_CHANGED）。
•	当输入为 "1234" 时，自动跳转到 Homepage。
•	主页（Homepage）
•	5 个图标按钮点击后分别跳转到：计算器、时钟、日历、闹钟、音乐界面。
•	日历（Calendar）
•	Back 按钮返回主页。
•	计算器（Calculator）
•	calculate_expression()：实现非常简化的表达式计算（整数 + - * / 整数，不支持多运算符链式/括号/小数）。
•	按钮矩阵事件：拼接输入、清空 C、等于 = 输出结果。
•	Back 返回主页。
•	时钟（Clock/Clockchange）
•	Clock：返回主页按钮；Setting 跳转到 Clockchange。
•	Clockchange：读取小时/分钟滚轮，更新全局 system_hour/system_min/system_sec，再返回 Clock。
•	闹钟（Alarmclock/Alarmclockchange）
•	Alarmclock：返回主页；Setting 跳转到闹钟设定页。
•	Alarmclockchange：读取滚轮，调用 save_alarm_time(new_hour,new_min) 保存闹钟时间，并尝试同步更新闹钟界面的数字时钟显示（Alarmclock_digital_clock_1）。
•	音乐播放器（Music）
•	基于蜂鸣器（BEEP_*）+ lv_timer 实现“按音符序列播放”：
•	music_start_play(idx)：开始播放曲目，创建定时器按音符时长推进。
•	music_play_next_note()：播放当前音符、更新进度条（按累积时长/总时长换算百分比）、更新下一次 tick 周期。
•	music_stop_play()：停止定时器与蜂鸣器，重置播放状态。
•	列表项点击选择曲目（3 首：idx 0~2）。
•	进度条拖动：按百分比换算到目标时间，定位音符索引并继续播放/仅更新进度。
---
gui_guider.c
用途： UI 启动与屏幕生命周期标志初始化。
主要实现：
•	ui_init_style()：对 lv_style_t 做 init/reset，供生成代码复用。
•	init_scr_del_flag()：将各 screen 的 *_del 标志置 true（项目用它来判断是否需要重新 setup_scr_*）。
•	setup_ui()：初始化标志，创建并加载初始界面 Codelock。
---
lv_dclock.c
用途： 自定义 LVGL 数字时钟控件实现（lv_dclock），并包含计时逻辑函数。
主要实现：
•	lv_dclock_create() / lv_dclock_set_text() / lv_dclock_set_text_fmt()：创建控件、设置显示文本。
•	LVGL 对象类回调：constructor/destructor/event/draw，负责按 label 方式绘制文本。
•	计时函数
•	clock_count_24(int *hour,int *min,int *sec)：
•	每次调用让全局 system_sec++，并做进位到分钟/小时（0~23）。
•	再把全局值写回参数（相当于“系统时间走秒”）。
•	clock_count_12(...)：12 小时制计时与 AM/PM 翻转（传统实现）。
注：源码中提到“修改了 clock_count_24”，说明此函数与项目全局时间变量耦合（非纯控件行为）。
---
music_data.c
用途： 音乐曲谱与曲目列表数据源。
主要实现：
•	定义三首曲子的 freqs[] 与 durations[]（单位 ms）。
•	MusicInfo g_music_list[]：曲目数组（包含音符序列指针、音符数、总时长）。
•	g_music_count：曲目数量。
•	供 events_init.c 的音乐播放逻辑使用。
---
2) 各界面 UI 构建（自动生成 + 少量项目定制）
下面这些 setup_scr_*.c 的共同点：创建 screen、放置控件、设置样式，最后调用对应的 events_init_XXX(ui) 绑定事件。重点交互逻辑一般不在这些文件里。
setup_scr_Codelock.c
•	创建“密码锁/登录”界面：标签、密码输入框（textarea）、键盘（可隐藏）。
•	事件绑定：events_init_Codelock()（密码正确跳转主页逻辑在 events_init.c）。
setup_scr_Homepage.c
•	创建主菜单界面：5 个图片按钮（计算器/时钟/日历/闹钟/音乐）。
•	事件绑定：events_init_Homepage()（点击跳转逻辑在 events_init.c）。
setup_scr_Calendar.c
•	创建日历界面：lv_calendar + header。
•	内含日历绘制回调（DRAW_PART_BEGIN）与日期选择高亮逻辑。
•	Back 事件在 events_init.c。
setup_scr_Calculator.c
•	创建计算器界面：按钮矩阵（数字 + 运算符 + C/=）与显示输入框（textarea）。
•	计算逻辑与按钮响应在 events_init.c。
setup_scr_Clock.c
•	创建时钟显示界面：使用 lv_dclock_create() 创建数字时钟控件。
•	创建 LVGL 定时器：每秒调用 Clock_digital_clock_1_timer()，内部调用 clock_count_24() 更新显示。
•	Back/Setting 按钮事件在 events_init.c。
setup_scr_Clockchange.c
•	创建时钟设置界面：小时/分钟滚轮，OK 按钮。
•	OK 事件在 events_init.c：读取滚轮并更新全局时间后返回 Clock。
setup_scr_Alarmclock.c
•	创建闹钟显示界面：数字时钟显示闹钟时间（初始化来自 alarm_info.hour/minute）。
•	包含“Stop”按钮（停止蜂鸣器），此按钮事件在本文件里直接绑定到 stop_buzzer()（同时该停止逻辑在 events_init.c 也有同名处理函数定义，但这里是本文件实际使用的绑定）。
•	Back/Setting 跳转在 events_init.c。
setup_scr_Alarmclockchange.c
•	创建闹钟设置界面：小时/分钟滚轮（默认选中 alarm_info 当前值）+ OK 按钮。
•	OK 事件在 events_init.c：保存闹钟时间并返回 Alarmclock。
setup_scr_Music.c
•	创建音乐播放器界面：曲目列表（3 项）、进度条 slider、返回按钮。
•	播放/拖动/返回逻辑在 events_init.c。
---
3) 图片资源（数据文件）
这些文件用于提供 LVGL 图片资源的像素数组（*_map[]），本身不包含业务逻辑：
•	_alarm_clock_icon_238228_alpha_60x60.c：闹钟图标资源
•	_calculatoroutlinedtool_81120_alpha_60x60.c：计算器图标资源
•	_calendar_date_event_schedule_icon_127191_alpha_60x60.c：日历图标资源
•	_clock_icon_143032_alpha_60x60.c：时钟图标资源
•	_vinyl_player_music_disc_turntable_icon_262851_alpha_60x60.c：音乐/唱片图标资源
---
4) 字体资源（数据文件）
这些 .c 用于定义 LVGL 字体（glyph bitmap + 字符映射表 + lv_font_t），不包含业务逻辑：
•	lv_font_montserratMedium_12.c
•	lv_font_montserratMedium_15.c
•	lv_font_montserratMedium_16.c
•	lv_font_montserratMedium_21.c
•	lv_font_montserratMedium_25.c
•	lv_font_simsun_18.c
---
交互流程（从代码可见）
1.	启动：setup_ui() 加载 Codelock。
2.	密码框输入到 "1234"：自动跳转 Homepage。
3.	Homepage 五个图标分别进入各功能：
•	计算器：按钮矩阵输入表达式 → = 计算
•	时钟：每秒刷新显示；可进入设置页修改系统时间
•	日历：可点选日期高亮；Back 返回
•	闹钟：显示已保存闹钟时间；设置页修改；Stop 停止蜂鸣器
•	音乐：点选曲目用蜂鸣器播放；进度条可拖动定位
