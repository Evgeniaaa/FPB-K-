# FPB-K-
#include <ncurses.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>

#define ESC_KEY          27

#define PIPE_X_GAP       30
#define PIPE_Y_GAP       10  // Фактическое значение между парами труб
#define PIPE_WIDTH       10
#define GRAVITY          1.0
#define SPEED            100
#define FLAPPY_X_POS     20

struct pipe_pair
{
    WINDOW *top_window;
    WINDOW *bottom_window;
    int pos_y;     // Центр разрыва труб

    int pos_x;     // Положение на оси х
};

struct bird
{
    WINDOW *window;
    int height;
    int score;
    int distance;
    float upward_speed;
};

char run_game(struct bird flappy, struct pipe_pair *pipes,
              unsigned int num_pipes);
void clear_pipes(struct pipe_pair *pipes, int num_pipes);
void welcome_message(int starty, int startx);
char death_message(int starty, int startx, unsigned int score);
WINDOW *draw_flappy_bird(int flappy_height, WINDOW *flappy_win);
void init_pipes(struct pipe_pair *pipes, unsigned int num_pipes,
                int lines, int coles);
int draw_pipes(struct pipe_pair *pipes, unsigned int num_pipes,
               int lines, int cols);

int main(void)
{
    unsigned int num_pipes;
    struct pipe_pair *pipes;
    struct bird flappy;
    
    initscr();// Переход в curses-режим,истит экран, выделяет память под необходимые данные для работы библиотеки
    cbreak();//позволяет управлять режимом клавиатуры
    noecho();//Выключаем отображение вводимых символов
    curs_set(0);  // Отключить отображение курсора
    keypad(stdscr, TRUE);// включает режим простой работы с функциональными клавишами,
    srand(time(NULL));//разные значения
    
    num_pipes = COLS/PIPE_X_GAP;
    
    pipes = (struct pipe_pair *) malloc(sizeof(struct pipe_pair) * num_pipes);
    
    refresh();
    welcome_message(LINES/2, COLS/2);
    
    while(run_game(flappy, pipes, num_pipes) != ESC_KEY)
        continue;
    
    free(pipes);
    refresh();
    endwin();
    return 0;
}

char run_game(struct bird flappy, struct pipe_pair *pipes, unsigned int num_pipes)
{
    char ch;
    int i;
    
    timeout(SPEED);
    
    init_pipes(pipes, num_pipes, LINES, COLS);
    
    flappy.height = LINES/2;
    flappy.upward_speed = 0;
    flappy.distance = 0;
    flappy.score = 0;
    flappy.window = draw_flappy_bird(flappy.height, NULL);
    draw_pipes(pipes, num_pipes, LINES, COLS);
    
    refresh();
    
    
    while((ch = getch()))
    {
        if(ch == ESC_KEY)
            goto death;
        
        else if(ch != ERR)
            flappy.upward_speed = 2.5;//скорость
        
        flappy.upward_speed -=GRAVITY;//горавитация
        
        if(flappy.upward_speed <= -2.0)
            flappy.upward_speed = -2.0;
        
        flappy.height -= flappy.upward_speed;
        
        
        flappy.window = draw_flappy_bird(flappy.height, flappy.window);
        flappy.score += draw_pipes(pipes, num_pipes, LINES, COLS);
        
        for(i = 0; i < num_pipes; i++)
        {
            if((pipes[i].pos_x - PIPE_WIDTH/2) <= FLAPPY_X_POS+2 &&
               (pipes[i].pos_x + PIPE_WIDTH/2) >= FLAPPY_X_POS)
            {
                if(flappy.height-4 <= (pipes[i].pos_y - PIPE_Y_GAP) ||
                   (flappy.height+4) >= (pipes[i].pos_y + PIPE_Y_GAP))
                { 
                    goto death;
                }
            }
        }
        
        if(flappy.height <= 0 || flappy.height >= LINES)
        {
            goto death;
        }
        mvprintw(LINES-1, 0, "Score: ");//счет
        attron(A_BOLD);
        printw("%d", flappy.score);
        attroff(A_BOLD);
    }
    
death:
    ch = death_message(LINES/2, COLS/2, flappy.score);
    clear_pipes(pipes, num_pipes);
    return ch;
}



int random_in_range(unsigned int min, unsigned int max)
{
    int base_random = rand();
    if(RAND_MAX == base_random) return random_in_range(min, max);
    
  
    int range       = max - min,
    remainder   = RAND_MAX % range,
    bucket      = RAND_MAX / range;
    
  
    if(base_random < RAND_MAX - remainder)
    {
        return min + base_random/bucket;
    }
    else
    {
        return random_in_range(min, max);
    }
}


void clear_pipes(struct pipe_pair *pipes, int num_pipes)
{
    int i;
    for(i = 0; i < num_pipes; i++)
    {
        werase(pipes[i].top_window);
        werase(pipes[i].bottom_window);
        wrefresh(pipes[i].top_window);
        wrefresh(pipes[i].bottom_window);
        delwin(pipes[i].top_window);
        delwin(pipes[i].bottom_window);
    }
}



int draw_pipes(struct pipe_pair *pipes, unsigned int num_pipes,
               int lines, int cols)
{
    int i;
    int passed = 0;
    clear_pipes(pipes, num_pipes);
    for(i = 0; i < num_pipes; i++)
    {
        pipes[i].pos_x -= 1;
        if(pipes[i].pos_x == (FLAPPY_X_POS - (PIPE_WIDTH/2) - 1))
        {
            passed = 1;
        }
        
        if(pipes[i].pos_x <= 0)
        {
            pipes[i].pos_x = cols + PIPE_X_GAP;
            pipes[i].pos_y = random_in_range(10, lines - 10);
        }
        pipes[i].top_window = newwin(
                                     pipes[i].pos_y - PIPE_Y_GAP/2,
                                     PIPE_WIDTH,
                                     0,
                                     pipes[i].pos_x - PIPE_WIDTH/2
                                     );
        
        pipes[i].bottom_window = newwin(
                                        lines - pipes[i].pos_y + PIPE_Y_GAP/2,
                                        PIPE_WIDTH,
                                        pipes[i].pos_y + PIPE_Y_GAP/2,
                                        pipes[i].pos_x - PIPE_WIDTH/2
                                        );
        box(pipes[i].top_window, 0, 0);
        box(pipes[i].bottom_window, 0, 0);
        wrefresh(pipes[i].top_window);
        wrefresh(pipes[i].bottom_window);
    }
    return passed;
}

void init_pipes(struct pipe_pair *pipes, unsigned int num_pipes,
                int lines, int cols)
{
    int i;
    for(i = 0; i < num_pipes; i++)
    {
        pipes[i].pos_x = cols/2 + (PIPE_X_GAP * i);
        pipes[i].pos_y = random_in_range(10, lines - 10);
        pipes[i].top_window = newwin(
                                     pipes[i].pos_y - PIPE_Y_GAP/2,
                                     PIPE_WIDTH,
                                     pipes[i].pos_y - (pipes[i].pos_y - PIPE_Y_GAP/2)/2,
                                     pipes[i].pos_x - PIPE_WIDTH/2
                                     );
        pipes[i].bottom_window = newwin(
                                        lines - pipes[i].pos_y - PIPE_Y_GAP/2,
                                        PIPE_WIDTH,
                                        pipes[i].pos_y - (pipes[i].pos_y - PIPE_Y_GAP/2)/2,
                                        pipes[i].pos_x - PIPE_WIDTH/2
                                        );
    }
}


WINDOW *draw_flappy_bird(int flappy_height, WINDOW *flappy_win)
{
    if(flappy_win != NULL)
    {
        werase(flappy_win);
        wrefresh(flappy_win);
        delwin(flappy_win);
    }
    flappy_win = newwin(3, 3, flappy_height, FLAPPY_X_POS);
    mvwprintw(flappy_win, 0, 0, " *");
    mvwprintw(flappy_win, 1, 0, "***");
  
    
    wrefresh(flappy_win);
    return flappy_win;
}

char death_message(int starty, int startx, unsigned int score)
{
    char ch;
    WINDOW *death_win;
    char *death_message[] = {
        "                 You died              ",
        "               Your score: ",
        "          Press  enter to try again   ",
        "                     or  ",
        "              Press ESC to quit ",
    };
    int width, height;
    
    timeout(-1);
    
    
    width = sizeof("Press ESC to quit or any button to try again") / sizeof(char);
    height = sizeof(death_message) / sizeof(death_message[0]);
    
    death_win = newwin(height + 2,
                       width + 1,
                       starty-(height/2),
                       startx-(width/2));
    
    mvwprintw(death_win, 1, 1, "%s", death_message[0]);
    mvwprintw(death_win, 2, 1, "%s", death_message[1]);
    wprintw(death_win, "%u", score);
    mvwprintw(death_win, 3, 1, "%s", death_message[2]);
    box(death_win, 0 , 0);
    wrefresh(death_win);
    
    while ((ch = getch()))
        if ((ch == ESC_KEY) || (ch == '\n'))
            break;
    
    
    werase(death_win);
    wrefresh(death_win);
    delwin(death_win);
    
    return ch;
}


void welcome_message(int starty, int startx)
{
    WINDOW *welcome_win;
    char *welcome_message[] = {
        "            Welcome to FPB",
        
        " Press ESC to quit or PRESS ANY KEY TO START ",
    };
    int width, height;
    
   
    width = sizeof("Escape to quit and any other key to jump") / sizeof(char);
    height = sizeof(welcome_message) / sizeof(welcome_message[0]);
    
    welcome_win = newwin(height + 2,
                         width + 1,
                         starty-(height/2),
                         startx-(width/2));
    mvwprintw(welcome_win, 1, 1, "%s", welcome_message[0]);
    mvwprintw(welcome_win, 2, 1, "%s", welcome_message[1]);
    mvwprintw(welcome_win, 3, 1, "%s", welcome_message[2]);
    box(welcome_win, 0 , 0);
    wrefresh(welcome_win);
    
    getch();
    
    werase(welcome_win);
    wrefresh(welcome_win);
    delwin(welcome_win);
}
