# perverseit[avalieh.py](https://github.com/user-attachments/files/24018192/avalieh.py)
# preserve_it_fixed.py
# ESC key disabled + fixed event handling + input box behavior

import pygame
import random
import sys
import time

pygame.init()

# -------------------- CONFIG --------------------
WIDTH, HEIGHT = 1200, 720
FPS = 60

GRID_SIZE = 5
CELL_SIZE = 80
GRID_ORIGIN = (60, 60)

RIGHT_PANEL_X = 520

TOTAL_LEVELS = 10
OPPORTUNITIES = 5

FILLED_PER_LEVEL = [1,2,2,3,3,4,4,5,5,6]

# Times
def memorize_time_for_level(l):
    if l <= 2: return 10
    if 3 <= l <= 5: return 7
    return 5

# Number digits
def number_range_for_level(l):
    if l <= 6: return (1,9)
    return (10,99)

# Colors
WHITE = (250,250,250)
BLACK = (10,10,10)
GRAY = (200,200,200)
DARKGRAY = (120,120,120)
GREEN = (30,180,70)
RED = (220,50,50)
BLUE = (60,120,200)
YELLOW = (230,200,60)

# -------------------- SETUP --------------------
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Preserve It")
clock = pygame.time.Clock()
font = pygame.font.SysFont(None, 28)
bigfont = pygame.font.SysFont(None, 42)
largefont = pygame.font.SysFont(None, 72)

# -------------------- SAVE STRUCTURE --------------------
save = []
for i in range(TOTAL_LEVELS):
    save.append({
        'solved': False,
        'circles': ['black'] * OPPORTUNITIES
    })
save[0]['circles'] = ['gray']*OPPORTUNITIES

# -------------------- UI HELPERS --------------------
def draw_text(s, x, y, surf=screen, f=font, color=BLACK):
    surf.blit(f.render(s, True, color), (x,y))

def rect_text_button(text, rect, color=GRAY, text_color=BLACK):
    pygame.draw.rect(screen, color, rect, border_radius=12)
    pygame.draw.rect(screen, BLACK, rect, 2, border_radius=12)
    tw, th = font.size(text)
    tx = rect[0] + (rect[2]-tw)//2
    ty = rect[1] + (rect[3]-th)//2
    draw_text(text, tx, ty, f=font, color=text_color)

# -------------------- GAME LOGIC --------------------
class LevelSession:
    def __init__(self, level_index):
        self.level_index = level_index
        self.level = level_index + 1
        self.memorize_time = memorize_time_for_level(self.level)
        self.filled_count = FILLED_PER_LEVEL[level_index]
        self.num_min, self.num_max = number_range_for_level(self.level)

        self.op_answers = []
        for _ in range(OPPORTUNITIES):
            coords = random.sample([(r,c) for r in range(GRID_SIZE) for c in range(GRID_SIZE)],
                                   self.filled_count)
            mapping = {coord: random.randint(self.num_min, self.num_max) for coord in coords}
            self.op_answers.append(mapping)

        self.temp_circles = ['gray']*OPPORTUNITIES
        self.current_op = 0
        self.greens = 0
        self.mode = 'ready'
        self.timer = 0.0
        self.guessed_positions = set()

    def start_opportunity(self):
        self.mode = 'preserving'
        self.timer = self.memorize_time
        self.guessed_positions = set()

    def hide_and_guess(self):
        self.mode = 'guessing'
        self.timer = 0.0

    def mark_correct_op(self):
        self.temp_circles[self.current_op] = 'green'
        self.greens += 1

    def mark_wrong_op(self):
        self.temp_circles[self.current_op] = 'red'

    def is_solved(self):
        return self.greens >= 3

# -------------------- INPUT BOX --------------------
class InputBox:
    def __init__(self, x,y,w,h, prompt=''):
        self.rect = pygame.Rect(x,y,w,h)
        self.text = ''
        self.active = True
        self.prompt = prompt

    def handle_event(self, event):
        if event.type == pygame.KEYDOWN and self.active:
            # ESC does nothing
            if event.key == pygame.K_ESCAPE:
                return None

            if event.key == pygame.K_BACKSPACE:
                self.text = self.text[:-1]
            elif event.key == pygame.K_RETURN:
                return 'submit'
            else:
                if event.unicode.isdigit() and len(self.text) < 3:
                    self.text += event.unicode
        return None

    def draw(self):
        pygame.draw.rect(screen, WHITE, self.rect)
        pygame.draw.rect(screen, BLACK, self.rect, 2)
        draw_text(self.prompt + self.text, self.rect.x+8, self.rect.y+8)



# -------------------- SCREENS --------------------
state = 'menu'  # 'menu', 'playing', 'message'
current_session = None
message_timer = 0.0
message_text = ''
message_next = None

def open_level(level_index):
    global state, current_session
    if save[level_index]['circles'][0] == 'black':
        return
    current_session = LevelSession(level_index)
    state = 'playing'
    current_session.start_opportunity()

def finalize_level_session(win):
    global state, current_session
    idx = current_session.level_index
    if win:
        save[idx]['solved'] = True
        save[idx]['circles'] = current_session.temp_circles.copy()
        if idx+1 < TOTAL_LEVELS:
            if save[idx+1]['circles'][0] == 'black':
                save[idx+1]['circles'] = ['gray']*OPPORTUNITIES
    else:
        save[idx]['solved'] = False
        save[idx]['circles'] = current_session.temp_circles.copy()
    current_session = None
    state = 'menu'

# -------------------- DRAWING HELPERS --------------------
def draw_main_menu():
    screen.fill(WHITE)
    draw_text('preserve it', 40, 10, f=largefont)
    draw_text('creator : behnia akbari', 40, 70)

    restart_rect = pygame.Rect(WIDTH-200, 20, 150, 45)
    rect_text_button('restart the game', restart_rect, color=GRAY)

    cols = 4
    box_w = 220
    box_h = 90
    start_x = 40
    start_y = 120
    gap_x = 30
    gap_y = 20

    for i in range(12):
        row = i // cols
        col = i % cols
        x = start_x + col*(box_w + gap_x)
        y = start_y + row*(box_h + gap_y)
        rect = pygame.Rect(x,y,box_w,box_h)

        pygame.draw.rect(screen, WHITE, rect, border_radius=16)
        pygame.draw.rect(screen, BLACK, rect, 2, border_radius=16)

        if i < TOTAL_LEVELS:
            draw_text(f'lvl {i+1}', x+12, y+10, f=bigfont)

            cx = x+12
            cy = y+52
            for j in range(OPPORTUNITIES):
                c = save[i]['circles'][j]
                if c == 'green': color = GREEN
                elif c == 'red': color = RED
                elif c == 'gray': color = GRAY
                elif c == 'black': color = BLACK
                else: color = DARKGRAY

                pygame.draw.circle(screen, color, (cx+30*j, cy), 10)
                pygame.draw.circle(screen, BLACK, (cx+30*j, cy), 10, 2)

        else:
            draw_text('coming soon...', x+20, y+32, f=bigfont)

    draw_text('version : 1.0.1', WIDTH-200, HEIGHT-50)
    return restart_rect

def draw_level_page(session):
    screen.fill(WHITE)
    draw_text('preserve it', 40, 10, f=largefont)

    grid_rects = []
    gx, gy = GRID_ORIGIN

    for r in range(GRID_SIZE):
        for c in range(GRID_SIZE):
            rect = pygame.Rect(gx + c*CELL_SIZE, gy + r*CELL_SIZE, CELL_SIZE-2, CELL_SIZE-2)
            pygame.draw.rect(screen, WHITE, rect)
            pygame.draw.rect(screen, BLACK, rect, 2)
            grid_rects.append(((r,c), rect))

    op_map = session.op_answers[session.current_op]

    if session.mode == 'preserving':
        for (r,c), val in op_map.items():
            x = gx + c*CELL_SIZE + 10
            y = gy + r*CELL_SIZE + 10
            draw_text(str(val), x, y, f=largefont, color=DARKGRAY)

    elif session.mode == 'guessing':
        for (r,c), val in op_map.items():
            if (r,c) in session.guessed_positions:
                x = gx + c*CELL_SIZE + 10
                y = gy + r*CELL_SIZE + 10
                draw_text(str(val), x, y, f=largefont, color=BLUE)

    exit_rect = pygame.Rect(RIGHT_PANEL_X+30, 20, 120, 42)
    rect_text_button('exit', exit_rect, color=GRAY, text_color=RED)

    if session.mode == 'preserving':
        draw_text(f'remaining time : {session.timer:0.2f}', RIGHT_PANEL_X+30, 90, f=bigfont)
    else:
        draw_text('remaining time :', RIGHT_PANEL_X+30, 90, f=bigfont)

    draw_text(f'level {session.level}', RIGHT_PANEL_X+30, 160)

    # situation
    sit_color = RED if session.mode == 'preserving' else GREEN
    sit_text = 'preserving' if session.mode == 'preserving' else 'guessing'
    txt_rect = pygame.Rect(RIGHT_PANEL_X+20, 200, 280, 50)
    rect_text_button(f'situation : {sit_text}', txt_rect, color=GRAY, text_color=sit_color)

    orect = pygame.Rect(RIGHT_PANEL_X+50, 270, 220, 40)
    rect_text_button('opportunity', orect, color=GRAY)

    cy = 350
    cx = RIGHT_PANEL_X+70
    for j in range(OPPORTUNITIES):
        c = session.temp_circles[j]
        if c == 'green': color = GREEN
        elif c == 'red': color = RED
        else: color = GRAY
        pygame.draw.circle(screen, color, (cx+50*j, cy), 18)
        pygame.draw.circle(screen, BLACK, (cx+50*j, cy), 18, 2)

    return exit_rect, grid_rects

# -------------------- MESSAGES --------------------
def show_message(text, duration=5, callback=None):
    global state, message_timer, message_text, message_next
    state = 'message'
    message_text = text
    message_timer = duration
    message_next = callback

# -------------------- MAIN LOOP --------------------
active_input = None
input_coord = None

while True:
    dt = clock.tick(FPS) / 1000.0
    events = pygame.event.get()

    # GLOBAL QUIT ONLY (ESC does nothing now)
    for ev in events:
        if ev.type == pygame.QUIT:
            pygame.quit()
            sys.exit()

    if state == 'menu':
        restart_rect = draw_main_menu()

        for ev in events:
            if ev.type == pygame.MOUSEBUTTONDOWN and ev.button == 1:
                mx,my = ev.pos

                # restart
                if restart_rect.collidepoint(mx,my):
                    def do_reset():
                        for i in range(TOTAL_LEVELS):
                            save[i]['solved'] = False
                            save[i]['circles'] = ['black']*OPPORTUNITIES
                        save[0]['circles'] = ['gray']*OPPORTUNITIES
                    show_message('Are you sure? Press Y or N', 1000, ('confirm_restart', do_reset))

                # level buttons
                cols = 4
                box_w = 220
                box_h = 90
                start_x = 40
                start_y = 120
                gap_x = 30
                gap_y = 20

                for i in range(12):
                    row = i // cols
                    col = i % cols
                    x = start_x + col*(box_w+gap_x)
                    y = start_y + row*(box_h+gap_y)
                    rect = pygame.Rect(x,y,box_w,box_h)

                    if rect.collidepoint(mx,my):
                        if i < TOTAL_LEVELS and save[i]['circles'][0] != 'black':
                            open_level(i)

        pygame.display.flip()

    elif state == 'playing':
        if current_session is None:
            state = 'menu'
            continue

        if current_session.mode == 'preserving':
            current_session.timer -= dt
            if current_session.timer <= 0:
                current_session.timer = 0
                current_session.hide_and_guess()

        exit_rect, grid_rects = draw_level_page(current_session)

        for ev in events:
            if ev.type == pygame.MOUSEBUTTONDOWN and ev.button == 1:
                mx,my = ev.pos

                if exit_rect.collidepoint(mx,my):
                    save[current_session.level_index]['circles'] = current_session.temp_circles.copy()
                    state = 'menu'
                    active_input = None
                    input_coord = None
                    break

                if current_session.mode == 'guessing':
                    clicked = None
                    for (coord, rect) in grid_rects:
                        if rect.collidepoint(mx,my):
                            clicked = coord
                            break

                    if clicked is not None:
                        op_map = current_session.op_answers[current_session.current_op]

                        if clicked not in op_map:
                            current_session.mark_wrong_op()
                            show_message('You lost this opportunity', 5, ('lost_op', None))
                            break

                        if clicked in current_session.guessed_positions:
                            break

                        if active_input is None:
                            active_input = InputBox(RIGHT_PANEL_X+40, 430, 220, 40, prompt='enter: ')
                            input_coord = clicked
                            break

            if ev.type == pygame.KEYDOWN:
                if state == 'message':
                    if message_next and message_next[0] == 'confirm_restart':
                        if ev.unicode.lower() == 'y':
                            message_next[1]()
                            state = 'menu'
                            message_next = None
                        elif ev.unicode.lower() == 'n':
                            state = 'menu'
                            message_next = None

        if active_input:
            for ev in events:
                res = active_input.handle_event(ev)

                if res == 'submit':
                    text = active_input.text

                    if text == 'menu':
                        save[current_session.level_index]['circles'] = current_session.temp_circles.copy()
                        state = 'menu'
                        active_input = None
                        input_coord = None
                        break

                    if not text.isdigit():
                        current_session.mark_wrong_op()
                        show_message('You lost this opportunity', 5, ('lost_op', None))
                        active_input = None
                        input_coord = None
                        break

                    val = int(text)
                    op_map = current_session.op_answers[current_session.current_op]

                    if input_coord not in op_map or val != op_map[input_coord]:
                        current_session.mark_wrong_op()
                        show_message('You lost this opportunity', 5, ('lost_op', None))
                        active_input = None
                        input_coord = None
                        break

                    current_session.guessed_positions.add(input_coord)
                    active_input = None
                    input_coord = None

                    if all(p in current_session.guessed_positions for p in op_map):
                        current_session.mark_correct_op()
                        show_message('You got everything correct!', 5, ('correct_op', None))

        if active_input:
            active_input.draw()

        pygame.display.flip()

    elif state == 'message':
        screen.fill(WHITE)
        draw_text(message_text, WIDTH//2 - 200, HEIGHT//2 - 30, f=bigfont)
        pygame.display.flip()

        for ev in events:
            if ev.type == pygame.KEYDOWN:
                if message_next and message_next[0] == 'confirm_restart':
                    if ev.unicode.lower() == 'y':
                        message_next[1]()
                        state = 'menu'
                        message_next = None
                    elif ev.unicode.lower() == 'n':
                        state = 'menu'
                        message_next = None

        message_timer -= dt
        if message_timer <= 0:
            if message_next and message_next[0] == 'lost_op':
                sess = current_session
                sess.current_op += 1
                if sess.current_op >= OPPORTUNITIES:
                    finalize_level_session(False)
                else:
                    sess.start_opportunity()
                    state = 'playing'

            elif message_next and message_next[0] == 'correct_op':
                sess = current_session
                if sess.is_solved():
                    finalize_level_session(True)
                else:
                    sess.current_op += 1
                    if sess.current_op >= OPPORTUNITIES:
                        finalize_level_session(sess.is_solved())
                    else:
                        sess.start_opportunity()
                        state = 'playing'
            else:
                state = 'menu'

            message_next = None
