# Hunter1410import pygame
import sys

# Constants
WIDTH, HEIGHT = 900, 600
FPS = 60
PLAYER_W, PLAYER_H = 40, 60
GRAVITY = 0.7
JUMP_VEL = -15
WALL_JUMP_VEL_X = 8
WALL_JUMP_VEL_Y = -13
COYOTE_TIME = 0.12  # seconds the player can still jump after leaving ground
JUMP_BUFFER = 0.12  # seconds jump input is buffered
MOVE_SPEED = 7
AIR_ACCEL = 0.5
WALL_SLIDE_SPEED = 3

pygame.init()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
clock = pygame.time.Clock()

class Player:
    def __init__(self):
        self.rect = pygame.Rect(100, HEIGHT - 120, PLAYER_W, PLAYER_H)
        self.vel_x = 0
        self.vel_y = 0
        self.on_ground = False
        self.on_wall_left = False
        self.on_wall_right = False
        self.coyote_timer = 0
        self.jump_buffer_timer = 0
        self.facing = 1
        self.jumping = False

    def handle_input(self, keys):
        move = 0
        if keys[pygame.K_LEFT]:
            move -= 1
        if keys[pygame.K_RIGHT]:
            move += 1
        self.facing = move if move != 0 else self.facing

        # Horizontal movement
        if self.on_ground:
            self.vel_x = move * MOVE_SPEED
        else:
            # Air control
            self.vel_x += move * AIR_ACCEL
            if abs(self.vel_x) > MOVE_SPEED:
                self.vel_x = MOVE_SPEED * (1 if self.vel_x > 0 else -1)

    def jump(self):
        if self.coyote_timer > 0 or self.on_wall_left or self.on_wall_right:
            if self.on_wall_left:
                self.vel_x = WALL_JUMP_VEL_X
                self.vel_y = WALL_JUMP_VEL_Y
            elif self.on_wall_right:
                self.vel_x = -WALL_JUMP_VEL_X
                self.vel_y = WALL_JUMP_VEL_Y
            else:
                self.vel_y = JUMP_VEL
            self.jumping = True
            self.coyote_timer = 0

    def apply_gravity(self, keys):
        # Variable jump height
        if self.jumping and not keys[pygame.K_SPACE] and self.vel_y < 0:
            self.vel_y += GRAVITY * 1.4
        else:
            self.vel_y += GRAVITY

        # Wall sliding
        if (self.on_wall_left or self.on_wall_right) and not self.on_ground and self.vel_y > 0:
            self.vel_y = min(self.vel_y, WALL_SLIDE_SPEED)

    def update_timers(self, dt):
        if self.on_ground:
            self.coyote_timer = COYOTE_TIME
        elif self.coyote_timer > 0:
            self.coyote_timer -= dt

        if self.jump_buffer_timer > 0:
            self.jump_buffer_timer -= dt

    def update(self, keys, platforms, dt):
        self.handle_input(keys)
        self.apply_gravity(keys)
        self.move_and_collide(platforms)
        self.update_timers(dt)

        # Jump buffering
        if self.jump_buffer_timer > 0:
            self.jump()
            self.jump_buffer_timer = 0

    def move_and_collide(self, platforms):
        # Horizontal move
        self.rect.x += int(self.vel_x)
        self.on_wall_left = self.on_wall_right = False
        for plat in platforms:
            if self.rect.colliderect(plat):
                if self.vel_x > 0:
                    self.rect.right = plat.left
                    self.on_wall_right = True
                    self.vel_x = 0
                elif self.vel_x < 0:
                    self.rect.left = plat.right
                    self.on_wall_left = True
                    self.vel_x = 0

        # Vertical move
        self.rect.y += int(self.vel_y)
        self.on_ground = False
        for plat in platforms:
            if self.rect.colliderect(plat):
                if self.vel_y > 0:
                    self.rect.bottom = plat.top
                    self.vel_y = 0
                    self.on_ground = True
                    self.jumping = False
                elif self.vel_y < 0:
                    self.rect.top = plat.bottom
                    self.vel_y = 0

    def draw(self, surf):
        color = (80, 220, 60) if self.on_ground else (100, 160, 255)
        pygame.draw.rect(surf, color, self.rect)
        # Draw simple face showing direction
        eye_x = self.rect.centerx + (10 * self.facing)
        pygame.draw.circle(surf, (0, 0, 0), (eye_x, self.rect.centery - 10), 4)

platforms = [
    pygame.Rect(0, HEIGHT - 40, WIDTH, 40),
    pygame.Rect(200, HEIGHT - 120, 180, 20),
    pygame.Rect(500, HEIGHT - 200, 200, 20),
    pygame.Rect(100, HEIGHT - 300, 120, 20),
    pygame.Rect(400, HEIGHT - 400, 24, 140),    # vertical wall
    pygame.Rect(700, HEIGHT - 350, 24, 150),    # wall for wall jumps
    pygame.Rect(650, HEIGHT - 500, 140, 24),    # high-up platform
]

player = Player()
while True:
    dt = clock.tick(FPS) / 1000
    keys = pygame.key.get_pressed()
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_SPACE:
                player.jump_buffer_timer = JUMP_BUFFER

    player.update(keys, platforms, dt)

    screen.fill((25, 24, 36))
    for plat in platforms:
        pygame.draw.rect(screen, (140, 140, 150), plat)
    player.draw(screen)
    pygame.display.flip()
