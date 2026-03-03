import pygame
import math

# --- Configuration ---
WIDTH, HEIGHT = 1200, 800
FPS = 60
WHITE = (255, 255, 255)
GREY = (50, 50, 50)
GREEN = (34, 139, 34)
RED = (255, 0, 0)

class Car:
    def __init__(self):
        self.x, self.y = WIDTH // 2, HEIGHT // 2
        self.angle = 0
        self.speed = 0
        self.max_speed = 8
        self.accel = 0.2
        self.friction = 0.05
        self.turn_speed = 4
        self.width, self.height = 20, 40

    def drive(self):
        keys = pygame.key.get_pressed()
        
        # Acceleration / Braking
        if keys[pygame.K_UP]:
            self.speed = min(self.speed + self.accel, self.max_speed)
        elif keys[pygame.K_DOWN]:
            self.speed = max(self.speed - self.accel, -self.max_speed / 2)
        else:
            # Natural deceleration (Friction)
            if self.speed > 0:
                self.speed = max(self.speed - self.friction, 0)
            elif self.speed < 0:
                self.speed = min(self.speed + self.friction, 0)

        # Steering (Only turns if moving)
        if self.speed != 0:
            direction = 1 if self.speed > 0 else -1
            if keys[pygame.K_LEFT]:
                self.angle += self.turn_speed * direction
            if keys[pygame.K_RIGHT]:
                self.angle -= self.turn_speed * direction

        # Physics: Convert angle/speed to X/Y coordinates
        vertical = math.cos(math.radians(self.angle)) * self.speed
        horizontal = math.sin(math.radians(self.angle)) * self.speed
        
        self.y -= vertical
        self.x -= horizontal

    def draw(self, surface):
        # Rotate car surface
        car_surf = pygame.Surface((self.width, self.height), pygame.SRCALPHA)
        pygame.draw.rect(car_surf, RED, (0, 0, self.width, self.height), border_radius=5)
        # Front lights to show direction
        pygame.draw.rect(car_surf, WHITE, (2, 0, 5, 5)) 
        pygame.draw.rect(car_surf, WHITE, (13, 0, 5, 5))
        
        rotated_surf = pygame.transform.rotate(car_surf, self.angle)
        rect = rotated_surf.get_rect(center=(self.x, self.y))
        surface.blit(rotated_surf, rect)

def main():
    pygame.init()
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    pygame.display.set_caption("Ultra-Realistic 2D Racer")
    clock = pygame.time.Clock()
    car = Car()

    running = True
    while running:
        screen.fill(GREEN) # Background Grass
        
        # Event handling
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False

        # Draw a basic circular "Track"
        pygame.draw.ellipse(screen, GREY, (100, 100, 1000, 600)) # Outer road
        pygame.draw.ellipse(screen, GREEN, (250, 250, 700, 300)) # Inner grass
        
        # Update Car
        car.drive()
        
        # Keep Camera Focused (Simple Boundary Check)
        # In a 2D script like this, we keep the car within screen limits
        car.x = max(20, min(WIDTH - 20, car.x))
        car.y = max(20, min(HEIGHT - 20, car.y))

        car.draw(screen)
        
        pygame.display.flip()
        clock.tick(FPS)

    pygame.quit()

if __name__ == "__main__":
    main()
