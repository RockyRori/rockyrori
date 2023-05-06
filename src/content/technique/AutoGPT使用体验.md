---
title: AutoGPT使用体验
publishDate: 2023-05-06 16:39:00
img: /assets/technique/AutoGPT使用体验/扫雷.jpg
img_alt: 
description: |
  文章中的图片可以在新窗口中打开，查看清晰版。
tags:
  - AI
  - AutoGPT
  - GPT
---


AutoGPT是一个利用ChatGPT生成能力的整合平台。ChatGPT是对话流，AutoGPT是一键流。它的愿景是输入目标，得到结果。例如，我想写一本书，我想画一幅画，我想做一个网站，这些是宏观的目标。如果是完全由自己完成这些事情，需要花费我们比较长的时间。如果是和ChatGPT对话，能节省大量的时间。如果是用AutoGPT来完成，你只需要告诉它你的目标，然后坐在旁边喝杯咖啡看它表演就好了。特别适合我这种终极懒癌患者。


AutoGPT的原理是代替我们和ChatGPT对话，相当于两个机器人互相聊天，然后把聊天记录实时打印给你看。我们要做的事情就是给它起一个聊天的话题。比如我想试试它的编程能力，我输入“快速的制作一个网页能够玩扫雷游戏”。


![输入目标](/assets/technique/AutoGPT使用体验/输入目标.png)


它把这个任务拆解成为了固定的身份+目标模式，这种方式特别值得我们学习，一个好的工科生应该具备超强的逻辑能力、任务拆解能力、执行能力。扯远点，我要飞上天这个目标是不现实的，但是把这个庞大的目标拆解成：学习飞在天上的生物的行为模式；研究空气动力学；能够承载高空低压的兼顾轻盈材料；等等一系列需求和任务，最终研制出双翼机就算是完成了我要飞上天这个伟大的目标了。平时工作中的模式也是制订一个能够交付客户使用的愿景、重大价值特性Epic，拆分成各个子模块、需求Feature交给团队负责人管理，在拆分成每人日完成独立完成的可量化的任务Story分派给各个一线员工。按照这套流程执行，一个重大的客户价值最终能够完成。


回到AutoGPT的工作流程。它给自己设定了一个身份：


Role: an autonomous agent that specializes in web development and assists users in creating custom web pages with various features and functionalities.角色：一个辅助机器人，专门从事web开发，帮助用户创建具有各种特性和功能的自定义web页面。


接着给自己设定了5个目标：Goals: 
- Develop a fully functional web page that allows users to play the classic game of Minesweeper with ease and efficiency.
- Provide customizable options for the game, such as different levels of difficulty and board sizes.
- Ensure the web page is user-friendly and visually appealing, with a clean and modern design.
- Optimize the web page for speed and performance, ensuring quick load times and smooth gameplay.
- Offer ongoing support and maintenance to ensure the web page remains functional and up-to-date.
- 开发一个功能齐全的网页，让用户轻松高效地玩经典的扫雷游戏。
- 为游戏提供可定制的选项，例如不同级别的难度和棋盘大小。
- 确保网页以简洁和现代的设计，方便使用者使用，视觉上吸引人。
- 优化网页的速度和性能，确保快速加载时间和流畅的游戏。
- 提供持续的支持和维护，以确保网页保持功能和最新。


设定了目标之后，AutoGPT根据自己的身份，按顺序逐一执行这些目标，执行过程中把结果回报出来，就像打工人写日报周报一样。同时，一个目标给能依旧难以完成，它的底层逻辑是用一个while true包裹起来，因此它会一直拆解任务执行任务直到最终达成用户输入的目标“快速的制作一个网页能够玩扫雷游戏”为止。


![拆解plan](/assets/technique/AutoGPT使用体验/拆解plan.png)


这次，它说我需要先分析《扫雷》游戏的需求。我还应该研究游戏的规则和玩法。一旦我对游戏有了很好的了解，我就可以开始计划网页的开发了。我还应该考虑不同的难度级别和玩家可以选择的棋盘大小。推理:分析需求和研究游戏规则将帮助我规划网页的开发。考虑到不同的难度和棋盘大小将使游戏更具可定制性。


你看，像不像一个人一样的思考？


并且给出了具体的五条计划，并且依次执行。

- 分析扫雷游戏的需求
- 研究游戏规则和游戏玩法
- 规划网页的开发
- 考虑不同的难度和棋盘大小


##### 第一步，它在本地创建了一个Python文件，minesweeper.py，写下了注释：
```
# Minesweeper game development plan
# Plan the development of the web page
# Decide on the layout of the game board, the different levels of difficulty, and the user interface
# Consider how to generate the game board and how to handle user input
```


##### 第二步，它更新了本地文件，添加了扫雷游戏后端需要的方法:
```
def generate_board(num_rows, num_cols, num_mines):
    pass

def check_game_status(board):
    pass

def handle_input(board, row, col, action):
    pass
```


##### 第三步，它说光有游戏还不行，为了保证游戏的功能正常运行我应该写一个测试文件：

- 创建一个测试文件，包括测试游戏板生成，用户输入处理和游戏状态检查
- 测试游戏，以确保它的功能并且没有bug


于是本地多出来一个minesweeper_test.py的文件：
```
import unittest
from minesweeper import generate_board, handle_input, check_game_status

class TestMinesweeper(unittest.TestCase):
    def test_generate_board(self):
        pass

    def test_handle_input(self):
        pass

    def test_check_game_status(self):
        pass
```


![创建测试文件](/assets/technique/AutoGPT使用体验/创建测试文件.png)


##### 第四步，测试文件已经创建好了，我该往里加点料了。于是它填充了测试用例：


```
import unittest
from minesweeper import generate_board, handle_input, check_game_status

class TestMinesweeper(unittest.TestCase):
    def test_generate_board(self):
        board = generate_board(5, 5)
        num_mines = sum(row.count(-1) for row in board)
        self.assertEqual(num_mines, 5)
        self.assertEqual(len(board), 5)
        self.assertEqual(len(board[0]), 5)

    def test_handle_input(self):
        board = [[0, 0, 0], [0, -1, 0], [0, 0, 0]]
        result = handle_input(board, 1, 1, 'flag')
        self.assertEqual(board[1][1], 'F')
        self.assertEqual(result, 'continue')
        result = handle_input(board, 1, 1, 'unflag')
        self.assertEqual(board[1][1], 0)
        self.assertEqual(result, 'continue')
        result = handle_input(board, 1, 1, 'reveal')
        self.assertEqual(board[1][1], 'X')
        self.assertEqual(result, 'lose')

    def test_check_game_status(self):
        board = [[0, 0, 0], [0, -1, 0], [0, 0, 0]]
        self.assertEqual(check_game_status(board), 'continue')
        board[1][1] = 'F'
        self.assertEqual(check_game_status(board), 'win')
        board[1][1] = 'X'
        self.assertEqual(check_game_status(board), 'lose')
        board = [[1, 1, 1], [1, -1, 1], [1, 1, 1]]
        self.assertEqual(check_game_status(board), 'win')

if __name__ == '__main__':
    unittest.main()
```

##### 第五步，执行测试用例看看程序写对了没？


![执行测试用例](/assets/technique/AutoGPT使用体验/执行测试用例.png)

![测试用例结果](/assets/technique/AutoGPT使用体验/测试用例结果.png)


好家伙，你猜怎么着，我本来以为机器写的代码可以跳过测试的，结果确实写错了。于是它说改进一下代码。重复修改代码和运行测试文件的过程持续了4轮，这里显得很不智能，最后是我看不下去了人为干预，检查它生成的代码和测试用例，修改了用例之后判定通过。


##### 第六步，修改程序代码以通过测试用例：

![改进程序代码](/assets/technique/AutoGPT使用体验/改进程序代码.png)

```
import random

def generate_board(num_rows, num_cols, num_mines):
    board = [[0 for j in range(num_cols)] for i in range(num_rows)]
    for i in range(num_mines):
        row = random.randint(0, num_rows - 1)
        col = random.randint(0, num_cols - 1)
        while board[row][col] == -1:
            row = random.randint(0, num_rows - 1)
            col = random.randint(0, num_cols - 1)
        board[row][col] = -1
        for r in range(row - 1, row + 2):
            for c in range(col - 1, col + 2):
                if r >= 0 and r < num_rows and c >= 0 and c < num_cols and board[r][c] != -1:
                    board[r][c] += 1
    return board

def check_game_status(board):
    if all(board[i][j] != 0 and board[i][j] != -1 for i in range(len(board)) for j in range(len(board[0]))):
        return 'win'
    return 'continue'

def handle_input(board, row, col, action):
    if action == 'flag':
        board[row][col] = 'F'
        return 'continue'
    elif action == 'unflag':
        board[row][col] = 0
        return 'continue'
    elif action == 'reveal':
        if board[row][col] == -1:
            return 'lose'
        elif board[row][col] == 'F':
            return 'continue'
        else:
            board[row][col] = 'X'
            return 'continue'
    return None
```

到此程序的后端部分已经能运行了，开始写前端。

##### 第七步，绘制前端页面：

我省略了中间的过程，综合起来，它分别写入了HTML、CSS、JavaScript，其实不尽如人意，中间出现了问题，AutoGPT陷入了查找Google，没有浏览器，没有GPT4.0的循环当中，我打断程序的运行。半成品页面是长这样的：

![前端页面](/assets/technique/AutoGPT使用体验/前端页面.png)


第八步，重启Agent，重新运行指令：

续上了之前关闭的任务，它开始写教程了，教会我们怎么用Python开发一个好的扫雷游戏。

![生产教程](/assets/technique/AutoGPT使用体验/生产教程.png)

因为教程太长了，竟然要花我2962 tokens，你知道什么概念吗？相当于写这篇文章花我14美元100人民币，吓死我了，赶紧结束任务。最终体验就到这里了。


![教程网页](/assets/technique/AutoGPT使用体验/教程网页.png)

##### 总结

AutoGPT既聪明又愚蠢，它现阶段还远远没有达到全自主阶段。但是它是非常好的辅助工具。就算是我自己要纯手写一个扫雷游戏的网页，其实说实话出Demo要一天时间，完整版要一周时间。如果用AutoGPT，可以在两天之内完成。当然这并不是最快的方法，仅仅在这个示例中，可以直接下载别人写好了的程序拿来用。但如果是创作一个原本不存在的事物呢？写一篇新的文章，它当然比我快。现在我还不至于被它压倒，或许再过一两年，AutoGPT就已经比我智能了。这并不意味着机器比人厉害，而是全世界最顶尖的人才创造的工具比全世界普通人厉害，所以还是人更厉害。以上。


