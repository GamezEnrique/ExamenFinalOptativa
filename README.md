import pygame
import os
import sys
import random

# Inicializar Pygame y el mixer de sonido
pygame.init()
pygame.mixer.init()

# Configuración de la pantalla
screen_width = 800
screen_height = 600
screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption("Skeleto Quest")

# Directorio de imágenes y sonidos
image_dir = os.path.join(os.path.dirname(__file__), 'images')
sound_dir = os.path.join(os.path.dirname(__file__), 'sounds')

# Verificar si el directorio de imágenes existe
if not os.path.exists(image_dir):
    print(f"Error: El directorio 'images' no se encuentra en {image_dir}")
    sys.exit()

# Función para cargar todas las imágenes en un diccionario
def load_images(directory):
    images = {}
    for filename in os.listdir(directory):
        if filename.endswith('.png'):
            image_path = os.path.join(directory, filename)
            image_key = os.path.splitext(filename)[0]
            images[image_key] = pygame.image.load(image_path).convert_alpha()
    return images

# Cargar todas las imágenes
images = load_images(image_dir)

# Verificar si el directorio de sonidos existe
if not os.path.exists(sound_dir):
    print(f"Error: El directorio 'sounds' no se encuentra en {sound_dir}")
    sys.exit()

# Cargar sonidos
pygame.mixer.music.load(os.path.join(sound_dir, 'background_music.mp3'))
game_over_sound = pygame.mixer.Sound(os.path.join(sound_dir, 'gameover.wav'))
collision_sound = pygame.mixer.Sound(os.path.join(sound_dir, 'collision.wav'))
level_up_sound = pygame.mixer.Sound(os.path.join(sound_dir, 'levelup.wav'))

# Asegurar que el archivo npcinteraction.wav exista en el directorio 'sounds'
npc_interaction_sound_path = os.path.join(sound_dir, 'npcinteraction.wav')
if os.path.exists(npc_interaction_sound_path):
    npc_interaction_sound = pygame.mixer.Sound(npc_interaction_sound_path)
else:
    default_sound_path = os.path.join(sound_dir, 'default_sound.wav')
    if os.path.exists(default_sound_path):
        npc_interaction_sound = pygame.mixer.Sound(default_sound_path)
    else:
        print("Error: No se encontraron los archivos de sonido 'npcinteraction.wav' o 'default_sound.wav'.")
        sys.exit()

enemy_defeated_sound = pygame.mixer.Sound(os.path.join(sound_dir, 'enemydefeated.mp3'))

# Usar la imagen del esqueleto para el sprite principal
skeleton_anim = {
    'down': [images.get(f'skeletonb{num}', None) for num in range(1, 3)],
    'up': [images.get(f'skeletont{num}', None) for num in range(1, 3)],
    'left': [images.get(f'skeletonl{num}', None) for num in range(1, 3)],
    'right': [images.get(f'skeletonr{num}', None) for num in range(1, 3)]
}

# Configuración del esqueleto
skeleton_rect = skeleton_anim['down'][0].get_rect()
skeleton_rect.topleft = (100, 100)
skeleton_speed = 5
direction = 'down'
anim_index = 0
player_health = 100  # Vida del jugador

# Fondo y niveles
levels = ['fondo2', 'fondoboss', 'icetree', 'Rockdesert']
current_level = 0
background = images.get(levels[current_level], None)
if background is None:
    print(f"Error: '{levels[current_level]}' image not found in the directory.")
    sys.exit()

# Crear NPCs
npcs = [
    {'image': images.get('Aldeano', None), 'rect': images.get('Aldeano', None).get_rect(topleft=(300, 300))},
    {'image': images.get('Orco', None), 'rect': images.get('Orco', None).get_rect(topleft=(500, 400))},
    {'image': images.get('Barbarian2', None), 'rect': images.get('Barbarian2', None).get_rect(topleft=(600, 150))}
]

# Crear enemigos
class Enemy(pygame.sprite.Sprite):
    def __init__(self, image, x, y, message):
        super().__init__()
        self.image = image
        self.rect = self.image.get_rect(topleft=(x, y))
        self.speed = 2
        self.direction = random.choice(['left', 'right', 'up', 'down'])
        self.shoot_timer = pygame.time.get_ticks()
        self.message = message

    def update(self):
        if self.direction == 'left':
            self.rect.x -= self.speed
        elif self.direction == 'right':
            self.rect.x += self.speed
        elif self.direction == 'up':
            self.rect.y -= self.speed
        elif self.direction == 'down':
            self.rect.y += self.speed

        # Cambiar de dirección al llegar al borde de la pantalla
        if self.rect.left < 0 or self.rect.right > screen_width:
            self.direction = 'right' if self.direction == 'left' else 'left'
        if self.rect.top < 0 or self.rect.bottom > screen_height:
            self.direction = 'down' if self.direction == 'up' else 'up'

        # Disparar balas
        current_time = pygame.time.get_ticks()
        if current_time - self.shoot_timer > 1000:  # Disparar cada 1 segundo
            bullet = Bullet(self.rect.centerx, self.rect.centery, self.direction)
            enemy_bullets.add(bullet)
            self.shoot_timer = current_time

# Crear más enemigos y agregarlos al grupo
enemies = pygame.sprite.Group()
enemy_positions = [
    (200, 200), (400, 300), (150, 450), (600, 200), (350, 350),
    (100, 100), (500, 150), (700, 300), (250, 500), (400, 100),
    (300, 400), (450, 250), (550, 150), (650, 300), (750, 200),
    (200, 500), (100, 350), (400, 400), (600, 450), (700, 100)  # Nuevas posiciones
]
enemy_images = [
    'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1',
    'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1',
    'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1',
    'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1', 'skeletonredb1', 'skeletongreenb1'  # Nuevas imágenes
]
enemy_messages = [
    "¡Te derrotaré!", "¡No escaparás!", "¡Prepárate para luchar!", "¡Eres mío!", "¡No tienes oportunidad!",
    "¡No me vas a vencer!", "¡Voy a acabar contigo!", "¡No tienes ninguna chance!", "¡Tu final ha llegado!", "¡Sufre!",
    "¡Rinde tus armas!", "¡No hay escapatoria!", "¡Siento tu miedo!", "¡Eres débil!", "¡Mi venganza es inevitable!",
    "¡Pagarás por tus crímenes!", "¡Te voy a atrapar!", "¡No tienes salvación!", "¡Es tu fin!", "¡No escaparás de mí!"  # Nuevos mensajes
]

for pos, img, msg in zip(enemy_positions, enemy_images, enemy_messages):
    enemies.add(Enemy(images.get(img, None), pos[0], pos[1], msg))

# Crear objetos coleccionables
collectibles = [
    {'image': images.get('Tesoro', None), 'rect': images.get('Tesoro', None).get_rect(topleft=(700, 500)), 'type': 'treasure'},
    {'image': images.get('espada', None), 'rect': images.get('espada', None).get_rect(topleft=(100, 500)), 'type': 'weapon'}
]

# Inicializar inventario
inventory = []

# Clase para las balas
class Bullet(pygame.sprite.Sprite):
    def __init__(self, x, y, direction):
        super().__init__()
        self.image = pygame.Surface((10, 5))
        self.image.fill((255, 255, 255))
        self.rect = self.image.get_rect()
        self.rect.center = (x, y)
        self.speed = 10
        self.direction = direction

    def update(self):
        if self.direction == 'left':
            self.rect.x -= self.speed
        elif self.direction == 'right':
            self.rect.x += self.speed
        elif self.direction == 'up':
            self.rect.y -= self.speed
        elif self.direction == 'down':
            self.rect.y += self.speed

        if self.rect.right < 0 or self.rect.left > screen_width or self.rect.bottom < 0 or self.rect.top > screen_height:
            self.kill()

# Grupo de sprites para las balas
bullets = pygame.sprite.Group()
enemy_bullets = pygame.sprite.Group()

# Configuración de fuentes
font = pygame.font.Font(None, 36)

# Variables de puntuación y tiempo
score = 0
high_score = 0  # Puntaje más alto
start_time = pygame.time.get_ticks()
enemy_message = ""  # Variable para almacenar el mensaje del enemigo

# Función para dibujar texto en la pantalla
def draw_text(text, font, color, surface, x, y):
    textobj = font.render(text, True, color)
    textrect = textobj.get_rect()
    textrect.center = (x, y)
    surface.blit(textobj, textrect)

# Función para mostrar pantalla de fin de juego
def show_end_screen(user_name, score, player_health, victory):
    screen.fill((0, 0, 0))  # Fondo negro
    font = pygame.font.Font(None, 74)
    
    if victory:
        title_surf = font.render("¡Has ganado!", True, (0, 255, 0))
    else:
        title_surf = font.render("¡Juego terminado!", True, (255, 0, 0))
    
    title_rect = title_surf.get_rect(center=(screen_width // 2, screen_height // 2 - 100))
    screen.blit(title_surf, title_rect)

    font = pygame.font.Font(None, 36)
    draw_text(f"Jugador: {user_name}", font, (255, 255, 255), screen, screen_width // 2, screen_height // 2)
    draw_text(f"Puntuación: {score}", font, (255, 255, 255), screen, screen_width // 2, screen_height // 2 + 40)
    draw_text(f"Vida: {player_health}", font, (255, 255, 255), screen, screen_width // 2, screen_height // 2 + 80)
    
    pygame.display.flip()
    pygame.time.wait(3000)  # Mostrar la pantalla durante 3 segundos

# Menú inicial
def main_menu():
    pygame.mixer.music.play(-1)
    screen.fill((0, 0, 0))  # Color negro como fondo

    font = pygame.font.Font(None, 32)
    input_box = pygame.Rect(0, 0, 400, 50)  # Rectángulo para la caja de entrada de texto
    input_box.center = (screen_width // 2, screen_height // 2)  # Centrar la caja de entrada de texto en la nueva resolución
    color_inactive = pygame.Color('lightskyblue3')
    color_active = pygame.Color('dodgerblue2')
    user_text = ''
    active = False
    start_game = False
    
    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                # Verificar clic en la caja de entrada de texto
                if input_box.collidepoint(event.pos):
                    active = not active
                else:
                    active = False
                # Verificar clic en el botón de iniciar juego
                start_button = pygame.Rect(0, 0, 300, 50)  # Rectángulo para el botón de iniciar juego
                start_button.center = (screen_width // 2, screen_height // 2 + 60)  # Posición del botón debajo de la caja de entrada de texto
                if start_button.collidepoint(event.pos):
                    start_game = True
            if event.type == pygame.KEYDOWN:
                if active:
                    if event.key == pygame.K_RETURN:
                        start_game = True
                    elif event.key == pygame.K_BACKSPACE:
                        user_text = user_text[:-1]
                    else:
                        user_text += event.unicode
        
        screen.fill((0, 0, 0))  # Limpiar la pantalla
        
        # Renderizar el título del juego
        title_font = pygame.font.Font(None, 74)
        title_surf = title_font.render("Juego del esqueleto", True, (255, 255, 255))
        title_rect = title_surf.get_rect(center=(screen_width // 2, screen_height // 2 - 100))
        screen.blit(title_surf, title_rect)
        
        # Renderizar la caja de entrada de texto
        color = color_active if active else color_inactive
        pygame.draw.rect(screen, color, input_box, 2)
        text_surface = font.render(user_text, True, (255, 255, 255))
        screen.blit(text_surface, (input_box.x + 5, input_box.y + 5))
        input_box.w = max(400, text_surface.get_width() + 10)
        
        # Renderizar el botón de iniciar juego
        start_button = pygame.Rect(0, 0, 300, 50)  # Rectángulo para el botón de iniciar juego
        start_button.center = (screen_width // 2, screen_height // 2 + 60)  # Posición del botón debajo de la caja de entrada de texto
        pygame.draw.rect(screen, (255, 255, 255), start_button)
        start_text = font.render("Iniciar Juego", True, (0, 0, 0))
        start_text_rect = start_text.get_rect(center=start_button.center)
        screen.blit(start_text, start_text_rect)
        
        if start_game:
            game_loop(user_text)  # Iniciar el juego pasando el nombre del usuario
            break
        
        pygame.display.flip()
        pygame.time.Clock().tick(30)

# Loop principal del juego
def game_loop(user_name):
    global direction, anim_index, current_level, background, score, high_score, enemy_message, player_health

    pygame.mixer.music.play(-1)

    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    running = False
                if event.key == pygame.K_SPACE:
                    bullet = Bullet(skeleton_rect.centerx, skeleton_rect.centery, direction)
                    bullets.add(bullet)

        keys = pygame.key.get_pressed()
        if keys[pygame.K_LEFT]:
            skeleton_rect.x -= skeleton_speed
            direction = 'left'
        if keys[pygame.K_RIGHT]:
            skeleton_rect.x += skeleton_speed
            direction = 'right'
        if keys[pygame.K_UP]:
            skeleton_rect.y -= skeleton_speed
            direction = 'up'
        if keys[pygame.K_DOWN]:
            skeleton_rect.y += skeleton_speed
            direction = 'down'

        anim_index = (anim_index + 1) % len(skeleton_anim[direction])

        screen.blit(background, (0, 0))

        for npc in npcs:
            screen.blit(npc['image'], npc['rect'])
            if skeleton_rect.colliderect(npc['rect']):
                npc_interaction_sound.play()
                # Mostrar mensaje del NPC
                draw_text("¡Hola, aventurero!", font, (255, 255, 255), screen, screen_width // 2, screen_height // 2)

        for collectible in collectibles:
            screen.blit(collectible['image'], collectible['rect'])
            if skeleton_rect.colliderect(collectible['rect']):
                if collectible['type'] == 'treasure':
                    level_up_sound.play()
                    score += 10
                elif collectible['type'] == 'weapon':
                    collision_sound.play()
                    inventory.append('espada')
                collectibles.remove(collectible)

        enemies.update()
        bullets.update()
        enemy_bullets.update()

        enemies.draw(screen)
        bullets.draw(screen)
        enemy_bullets.draw(screen)

        for enemy in enemies:
            if skeleton_rect.colliderect(enemy.rect):
                collision_sound.play()
                player_health -= 10
                enemy_message = enemy.message
                enemies.remove(enemy)
                if player_health <= 0:
                    game_over_sound.play()
                    show_end_screen(user_name, score, player_health, victory=False)
                    return

        for bullet in bullets:
            for enemy in enemies:
                if bullet.rect.colliderect(enemy.rect):
                    enemy_defeated_sound.play()
                    score += 5
                    bullets.remove(bullet)
                    enemies.remove(enemy)

        for bullet in enemy_bullets:
            if bullet.rect.colliderect(skeleton_rect):
                collision_sound.play()
                player_health -= 5
                enemy_bullets.remove(bullet)
                if player_health <= 0:
                    game_over_sound.play()
                    show_end_screen(user_name, score, player_health, victory=False)
                    return

        # Verificar si todos los enemigos han sido derrotados
        if not enemies:
            if score >= 50:
                current_level = (current_level + 1) % len(levels)
                background = images.get(levels[current_level], None)
                if background is None:
                    print(f"Error: '{levels[current_level]}' image not found in the directory.")
                    sys.exit()
                score = 0
                player_health = 100
                draw_text(f"¡{user_name}, has ganado!", font, (255, 255, 0), screen, screen_width // 2, screen_height // 2)
                pygame.display.flip()
                pygame.time.wait(2000)
                show_end_screen(user_name, score, player_health, victory=True)
                return

        # Mostrar nombre del jugador, puntuación y vida en la parte superior central
        draw_text(f"Jugador: {user_name}", font, (255, 255, 255), screen, screen_width // 2, 20)
        draw_text(f"Puntuación: {score}", font, (255, 255, 255), screen, screen_width // 2, 60)
        draw_text(f"Vida: {player_health}", font, (255, 255, 255), screen, screen_width // 2, 100)
        draw_text(f"Mensaje: {enemy_message}", font, (255, 0, 0), screen, screen_width // 2, screen_height - 30)

        screen.blit(skeleton_anim[direction][anim_index], skeleton_rect)

        pygame.display.flip()
        pygame.time.Clock().tick(30)

main_menu()
