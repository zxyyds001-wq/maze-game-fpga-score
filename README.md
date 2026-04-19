# maze-game-fpga-score
README
Maze Game + FPGA Score Display
Overview

This project connects software logic with digital hardware.
I built a console maze game in C/C++ and used Verilog on an FPGA to display the player’s score for each game session.

What I built
Maze game (C/C++): text-based maze using a 2D array, player movement, and randomized traps/rewards
Score display (Verilog/FPGA): logic to show the final score for each completed game session
Key skills
C/C++ programming (arrays, control flow, debugging)
Verilog / FPGA basic design
Step-by-step troubleshooting across software and hardware
Simple testing and documentation
How to run (software part)
Compile and run the C/C++ maze game in a local environment
Follow the on-screen instructions to move the player and finish one session
FPGA part
The FPGA displays the final score after each game session (Verilog implemented by me)
Notes / next steps
Improve test coverage for edge cases (walls, repeated events, score consistency)
Add clearer documentation and screenshots/photos of the FPGA output

Project Code:
#include <iostream>
#include <cstdlib> // for srand() and rand()
#include <ctime>   // for time()
#include <string>
#include <cctype>  // toupper
#define ROWS 10
#define COLS 20

const int HP_START       = 100;  // initialize the starting HP to 100
const int HP_STEP        = -1;   // 每走一步扣 1 HP
const int HP_TRAP        = -10;  // 踩陷阱额外扣 10 HP
const int HP_TREASURE    = 10;   // 宝藏回 10 HP
const int SCORE_TREASURE = 20;   // 宝藏得 20 分

const int TRAP_PERCENT     = 10; // 10% 概率放陷阱
const int TREASURE_PERCENT = 10; // 10% 概率放宝藏

const int PADDING   = 4;  // 左侧缩进，让整块地图不贴左边
const int CELL_SPAN = 2;  // let each block has 2 bytes width to make sure easy to read

/* function prototypes */
void initVisibleMap(char visibleMap[][COLS], int playerRow, int playerCol, int exitRow, int exitCol);
void initFullMap(char fullMap[][COLS], int playerRow, int playerCol, int &exitRow, int &exitCol);
void printBoard(char visibleMap[][COLS], int hp, int score, int steps, const std::string& msg);
void clearScreen();
bool inBounds(int r, int c);
void applyTile(char fullMap[][COLS], char visibleMap[][COLS],
               int r, int c, int &hp, int &score, bool &win, std::string &msg);

int main(int argc, char const *argv[])
{
    std::srand((unsigned)std::time(NULL)); // random seed

    char fullMap[ROWS][COLS];     // actual map with trap(T) and treasure($) that user cannot see
    char visibleMap[ROWS][COLS];  // the map visible for user

    // initialize the player space (editable)
    int playerRow = ROWS - 2;  // 你可以随便改起点
    int playerCol = 2;

    int exitRow = 0, exitCol = 0; // exit place（let initFullMap generate randomly）

    initFullMap(fullMap, playerRow, playerCol, exitRow, exitCol); // initialize the actual map
    initVisibleMap(visibleMap, playerRow, playerCol, exitRow, exitCol); // initialize the visible map

    /* player status */
    int hp    = HP_START; // starting hp value to 100
    int score = 0;        // score earned
    int steps = 0;        // steps

    bool win  = false; // win, lost, quit
    bool lost = false;
    bool quit = false;

    std::string msg =
        "You wake up in a dark dungeon. Reach the exit (E) before HP runs out!\n"
        "Rules: Each step costs 1 HP. Trap(T) costs extra 10 HP. "
        "Treasure($) gives +10 HP and +20 score.";

    // main loop of the game
    while(true)
    {
        clearScreen();
        printBoard(visibleMap, hp, score, steps, msg);

        if (win || lost || quit) break;

        std::string cmd; // player command string
        std::cin >> cmd;
        if(!std::cin) break;

        msg.clear(); // 每一轮先清空消息（之后根据操作更新）

        for(int i = 0; i < static_cast<int>(cmd.size()); i++)
        {
            char ch = cmd[i];
            ch = static_cast<char>(std::toupper(static_cast<unsigned char>(ch)));

            if (ch == ' ') continue;

            // quit
            if (ch == 'Q')
            {
                quit = true;
                msg = "You chose to quit.";
                break;
            }

            // invalid input -> ignore, no HP cost, no move
            if (ch != 'W' && ch != 'A' && ch != 'S' && ch != 'D')
            {
                msg = std::string("Ignored invalid input: ") + cmd[i];
                continue;
            }

            // compute next position
            int nextRow = playerRow;
            int nextCol = playerCol;
            if (ch == 'W') nextRow--;
            if (ch == 'S') nextRow++;
            if (ch == 'A') nextCol--;
            if (ch == 'D') nextCol++;

            // hit wall -> ignore move, no HP cost
            if (!inBounds(nextRow, nextCol))
            {
                msg = "You hit a wall. That move was ignored.";
                continue;
            }

            // -------- valid move begins --------

            // 1) 把当前格子“永久揭示”为真实内容（一般是 '.' 或 'T'）
            //    但如果当前格子是 P，我们要用 fullMap 的内容替换回去
            char curTile = fullMap[playerRow][playerCol];
            if (curTile == '$') curTile = '.'; // 理论上宝藏踩过就会变 '.'，保险
            visibleMap[playerRow][playerCol] = curTile;

            // 2) 扣每步 HP、步数+1
            hp += HP_STEP;  // HP_STEP = -1
            steps++;

            // 3) 移动玩家
            playerRow = nextRow;
            playerCol = nextCol;

            // 4) 结算新格子并揭示
            applyTile(fullMap, visibleMap, playerRow, playerCol, hp, score, win, msg);

            // 5) 如果还没结束，就把玩家标到 visibleMap 上
            //    （站在格子上时显示 P）
            if (!win && hp > 0)
                visibleMap[playerRow][playerCol] = 'P';

            // 6) 检查死亡
            if (hp <= 0)
            {
                lost = true;
                msg = "Game Over. You lost all your health.";
                break;
            }

            // 批量输入中途结束
            if (win) break;
        }
    }

    clearScreen();
    printBoard(visibleMap, hp, score, steps, msg);

    if (win)  std::cout << "\nVictory! You escaped the dungeon!\n";
    if (lost) std::cout << "\nGame Over. You died in the dungeon.\n";
    if (quit) std::cout << "\nQuit. See you next time.\n";

    std::cout << "Final Score: " << score << "\n";
    std::cout << "Total Steps: " << steps << "\n";

    return 0;
}

/* helper functions (function definitions) */

// initialize the visibleMap：all black spaces，only print P and E
void initVisibleMap(char visibleMap[][COLS], int playerRow, int playerCol, int exitRow, int exitCol)
{
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            visibleMap[r][c] = ' ';   // unexplored spaces are represented by space

    visibleMap[playerRow][playerCol] = 'P';
    visibleMap[exitRow][exitCol]     = 'E';
}

void initFullMap(char fullMap[][COLS], int playerRow, int playerCol, int &exitRow, int &exitCol)
{
    for(int r = 0; r < ROWS; r++)
        for (int c = 0 ; c < COLS; c++)
            fullMap[r][c] = '.'; // safe place

    // random exit(E) not overlapping with player(P)
    do
    {
        exitRow = std::rand() % ROWS;
        exitCol = std::rand() % COLS;
    } while (exitRow == playerRow && exitCol == playerCol);

    fullMap[exitRow][exitCol] = 'E';

    // random trap(T) and treasure($), exclude overlapping with P and E
    for(int r = 0; r < ROWS; r++)
    {
        for(int c = 0; c < COLS; c++)
        {
            if((r == playerRow && c == playerCol) || (r == exitRow && c == exitCol))
                continue;

            int roll = std::rand() % 100; // 0 ~ 99

            if(roll < TRAP_PERCENT)
                fullMap[r][c] = 'T';
            else if(roll < TRAP_PERCENT + TREASURE_PERCENT)
                fullMap[r][c] = '$';
        }
    }
}

bool inBounds(int r, int c)
{
    return r >= 0 && r < ROWS && c >= 0 && c < COLS;
}

// apply effect of current tile then reveal it on visibleMap
void applyTile(char fullMap[][COLS], char visibleMap[][COLS],
               int r, int c, int &hp, int &score, bool &win, std::string &msg)
{
    char tile = fullMap[r][c];

    if (tile == 'E')
    {
        win = true;
        visibleMap[r][c] = 'E';
        msg = "You reached the exit!";
        return;
    }

    if (tile == '.')
    {
        visibleMap[r][c] = '.';
        msg = "Safe tile.";
    }
    else if (tile == 'T')
    {
        visibleMap[r][c] = 'T';
        hp += HP_TRAP; // HP_TRAP = -10
        msg = "Trap! -10 HP.";
    }
    else if (tile == '$')
    {
        hp += HP_TREASURE;           // +10 HP
        score += SCORE_TREASURE;     // +20 score

        fullMap[r][c] = '.';         // treasure becomes safe after collected
        visibleMap[r][c] = '.';      // show safe on visible map

        msg = "Treasure found! +10 HP, +20 Score.";
    }
}

void printBoard(char visibleMap[][COLS], int hp, int score, int steps, const std::string& msg)
{
    std::string pad(PADDING, ' ');
    int innerWidth = COLS * CELL_SPAN;

    std::cout << "\n\n";
    if (!msg.empty()) std::cout << pad << msg << "\n\n";

    // upper edge
    std::cout << pad;
    for (int i = 0; i < innerWidth + 2; i++)
        std::cout << '#';
    std::cout << '\n';

    // rows
    for (int r = 0; r < ROWS; r++)
    {
        std::cout << pad << '#';

        for (int c = 0; c < COLS; c++)
            std::cout << ' ' << visibleMap[r][c];

        std::cout << "#\n";
    }

    // lower edge
    std::cout << pad;
    for (int i = 0; i < innerWidth + 2; i++)
        std::cout << '#';
    std::cout << "\n\n";

    // HUD
    std::cout << pad << "HP: "    << hp
              << "   Score: "     << score
              << "   Steps: "     << steps << "\n";
    std::cout << pad << "Next move [W/A/S/D or Q]: _\n";
}

// clear the screen：Windows: cls, Mac and Linux: clear
void clearScreen()
{
#ifdef _WIN32
    std::system("cls");
#else
    std::system("clear");
#endif
}
