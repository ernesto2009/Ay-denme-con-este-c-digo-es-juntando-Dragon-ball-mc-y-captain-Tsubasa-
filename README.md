# Ay-denme-con-este-c-digo-es-juntando-Dragon-ball-mc-y-captain-Tsubasa-




import pygame
import math
import random

# Inicialización
pygame.init()
pygame.mixer.init()

ANCHO, ALTO = 1100, 650
pantalla = pygame.display.set_mode((ANCHO, ALTO))
pygame.display.set_caption("Block Strikers: Ki World ⚽⚡🧱")
reloj = pygame.time.Clock()

def reproducir_sonido(freq, duracion, tipo='sine'):
    try:
        sample_rate = 22050
        n_samples = int(sample_rate * (duracion / 1000.0))
        buf = bytearray()
        for i in range(n_samples):
            t = float(i) / sample_rate
            if tipo == 'sine': 
                val = int(127 + 127 * math.sin(2 * math.pi * freq * t))
            elif tipo == 'square': 
                val = 255 if math.sin(2 * math.pi * freq * t) > 0 else 0
            elif tipo == 'noise': 
                val = random.randint(0, 255)
            buf.append(max(0, min(255, val)))
        sonido = pygame.mixer.Sound(buffer=bytes(buf))
        sonido.set_volume(0.15)
        sonido.play()
    except Exception: 
        pass

# --- CONSTANTES Y COLORES ---
TAMANO_BLOQUE = 32
COLOR_CIELO = (30, 35, 55)
COLOR_PASTO = (46, 204, 113)
COLOR_TIERRA = (120, 66, 18)
COLOR_PIEDRA = (127, 140, 141)
COLOR_MADERA = (141, 110, 99)
COLOR_HOJAS = (39, 174, 96)
COLOR_OBSIDIANA = (44, 62, 80)

COLOR_HEROE = (230, 126, 34)
COLOR_PELO_BASE = (30, 30, 30)
COLOR_PELO_SSJ = (241, 196, 15)
COLOR_KI = (52, 152, 219)
COLOR_FUEGO = (231, 76, 60)

TIPOS_BLOQUES = {
    1: {"nombre": "Tierra/Pasto", "color": COLOR_TIERRA, "hp": 1},
    2: {"nombre": "Piedra", "color": COLOR_PIEDRA, "hp": 3},
    3: {"nombre": "Madera", "color": COLOR_MADERA, "hp": 2},
    4: {"nombre": "Hojas", "color": COLOR_HOJAS, "hp": 1},
    5: {"nombre": "Obsidiana", "color": COLOR_OBSIDIANA, "hp": 999}
}

# --- GENERACIÓN DEL MUNDO LIBRE (Procedural) ---
mapa = {}
ANCHO_MUNDO = 2000

def generar_mundo():
    for x in range(-500, ANCHO_MUNDO, TAMANO_BLOQUE):
        # Generar relieve con ondas senoidales
        altura_suelo = int(ALTO - 120 + math.sin(x * 0.01) * 40)
        
        # Bloque de superficie
        mapa[(x, altura_suelo)] = {"tipo": 1, "hp": 1}
        
        # Capas subterráneas
        for y in range(altura_suelo + TAMANO_BLOQUE, ALTO + 300, TAMANO_BLOQUE):
            mapa[(x, y)] = {"tipo": 2, "hp": 3}
            
        # Árboles aleatorios
        if random.random() < 0.08 and x > 100:
            tronco_h = random.randint(3, 5)
            for h in range(1, tronco_h + 1):
                mapa[(x, altura_suelo - h * TAMANO_BLOQUE)] = {"tipo": 3, "hp": 2}
            # Copa de hojas
            top_y = altura_suelo - (tronco_h + 1) * TAMANO_BLOQUE
            for hx in range(-TAMANO_BLOQUE, TAMANO_BLOQUE * 2, TAMANO_BLOQUE):
                for hy in range(-TAMANO_BLOQUE, TAMANO_BLOQUE, TAMANO_BLOQUE):
                    mapa[(x + hx, top_y + hy)] = {"tipo": 4, "hp": 1}

generar_mundo()

# ENTIDADES
jugador = {
    "rect": pygame.Rect(100, 200, 28, 48),
    "vx": 0, "vy": 0,
    "en_suelo": False,
    "mirando_derecha": True,
    "ki": 50, "max_ki": 100,
    "es_ssj": False,
    "bloque_sel": 1
}

balon = {
    "rect": pygame.Rect(200, 150, 20, 20),
    "vx": 0, "vy": 0,
    "tipo_tiro": None,
    "estela": []
}

rafagas_ki = []
particulas = []
camara_x, camara_y = 0, 0

cargando_tiro = False
tecla_tiro = None
potencia_tiro = 0.0
dir_potencia = 1.5
temp_cinematica = 0
datos_cinematica = {}

def activar_cinematica(nombre, perfecto):
    global temp_cinematica, datos_cinematica
    temp_cinematica = 40
    datos_cinematica = {
        "nombre": nombre, 
        "perfecto": perfecto, 
        "color": COLOR_FUEGO if "TIGRE" in nombre or "DEVASTADOR" in nombre else COLOR_KI
    }
    reproducir_sonido(100, 500, 'noise')

def crear_particulas(x, y, color, cant=5):
    for _ in range(cant):
        particulas.append({
            "x": x, "y": y, 
            "vx": random.uniform(-3, 3), 
            "vy": random.uniform(-3, 3), 
            "color": color, 
            "vida": 20
        })

# --- BUCLE DE JUEGO ---
jugando = True
cargando_ki = False

while jugando:
    dt = reloj.tick(60)

    # 1. CONTROLES Y EVENTOS
    for evento in pygame.event.get():
        if evento.type == pygame.QUIT: 
            jugando = False

        if evento.type == pygame.KEYDOWN and temp_cinematica == 0:
            if (evento.key in (pygame.K_SPACE, pygame.K_w)) and jugador["en_suelo"]:
                jugador["vy"] = -13 if jugador["es_ssj"] else -10.5
                reproducir_sonido(300, 60, 'square')

            if evento.key == pygame.K_u: # Transformación SSJ
                if not jugador["es_ssj"] and jugador["ki"] >= 70:
                    jugador["es_ssj"] = True
                    jugador["ki"] -= 20
                    crear_particulas(jugador["rect"].centerx, jugador["rect"].centery, COLOR_PELO_SSJ, 20)
                else: 
                    jugador["es_ssj"] = False

            if evento.key == pygame.K_b: # Invocación de Balón
                balon["rect"].x = jugador["rect"].x + 40
                balon["rect"].y = jugador["rect"].y - 20
                balon["vx"], balon["vy"] = 0, 0

            # Selección de bloques (1 a 5)
            if evento.key in [pygame.K_1, pygame.K_2, pygame.K_3, pygame.K_4, pygame.K_5]:
                jugador["bloque_sel"] = int(evento.unicode)

            if evento.key == pygame.K_j and jugador["ki"] >= 10: # Disparo de Ki
                jugador["ki"] -= 10
                dx = 1 if jugador["mirando_derecha"] else -1
                rafagas_ki.append({
                    "rect": pygame.Rect(jugador["rect"].centerx, jugador["rect"].centery - 6, 16, 12), 
                    "vx": 15 * dx, 
                    "color": COLOR_PELO_SSJ if jugador["es_ssj"] else COLOR_KI
                })

            dist_balon = math.hypot(jugador["rect"].centerx - balon["rect"].centerx, jugador["rect"].centery - balon["rect"].centery)
            if dist_balon < 60 and not cargando_tiro:
                if evento.key == pygame.K_k and jugador["ki"] >= 30:
                    cargando_tiro, tecla_tiro, potencia_tiro = True, 'k', 0.0
                elif evento.key == pygame.K_i and jugador["ki"] >= 35:
                    cargando_tiro, tecla_tiro, potencia_tiro = True, 'i', 0.0

        if evento.type == pygame.KEYUP and cargando_tiro:
            if (evento.key == pygame.K_k and tecla_tiro == 'k') or (evento.key == pygame.K_i and tecla_tiro == 'i'):
                perf = 75 <= potencia_tiro <= 95
                mult = 1.4 if perf else (potencia_tiro / 50.0)
                dx = 1 if jugador["mirando_derecha"] else -1

                if tecla_tiro == 'k':
                    jugador["ki"] -= 30
                    balon["vx"], balon["vy"], balon["tipo_tiro"] = (22 * mult) * dx, -3 * mult, "tigre"
                    activar_cinematica("¡TIRO DEVASTADOR!", perf)
                elif tecla_tiro == 'i':
                    jugador["ki"] -= 35
                    balon["vx"], balon["vy"], balon["tipo_tiro"] = (15 * mult) * dx, -17 * mult, "parabola"
                    activar_cinematica("¡TIRO CON ELEVACIÓN!", perf)

                cargando_tiro = False

        # CONSTRUCCIÓN Y DESTRUCCIÓN EN EL MUNDO
        if evento.type == pygame.MOUSEBUTTONDOWN and temp_cinematica == 0:
            mx, my = pygame.mouse.get_pos()
            world_x = (mx + camara_x) // TAMANO_BLOQUE * TAMANO_BLOQUE
            world_y = (my + camara_y) // TAMANO_BLOQUE * TAMANO_BLOQUE
            celda = (world_x, world_y)

            if evento.button == 1 and celda in mapa: # Clic izquierdo: Romper
                mapa[celda]["hp"] -= 1
                if mapa[celda]["hp"] <= 0: 
                    del mapa[celda]
            elif evento.button == 3 and celda not in mapa: # Clic derecho: Colocar
                r_b = pygame.Rect(world_x, world_y, TAMANO_BLOQUE, TAMANO_BLOQUE)
                if not r_b.colliderect(jugador["rect"]) and not r_b.colliderect(balon["rect"]):
                    mapa[celda] = {"tipo": jugador["bloque_sel"], "hp": TIPOS_BLOQUES[jugador["bloque_sel"]]["hp"]}

    # 2. LÓGICA DE JUEGO Y CÁMARA
    if cargando_tiro:
        potencia_tiro += dir_potencia * 3.5
        if potencia_tiro >= 100 or potencia_tiro <= 0: 
            dir_potencia *= -1

    if temp_cinematica == 0:
        teclas = pygame.key.get_pressed()
        vel_m = 1.5 if jugador["es_ssj"] else 1.0

        jugador["vx"] = 0
        if teclas[pygame.K_a] and not cargando_tiro: 
            jugador["vx"] = -5 * vel_m; jugador["mirando_derecha"] = False
        if teclas[pygame.K_d] and not cargando_tiro: 
            jugador["vx"] = 5 * vel_m; jugador["mirando_derecha"] = True
        if teclas[pygame.K_l] and not cargando_tiro: 
            jugador["ki"] = min(jugador["max_ki"], jugador["ki"] + 0.8)

        # Físicas del Jugador
        jugador["vy"] += 0.6
        jugador["rect"].x += jugador["vx"]
        for (bx, by), d_b in mapa.items():
            r_b = pygame.Rect(bx, by, TAMANO_BLOQUE, TAMANO_BLOQUE)
            if jugador["rect"].colliderect(r_b):
                if jugador["vx"] > 0: jugador["rect"].right = r_b.left
                elif jugador["vx"] < 0: jugador["rect"].left = r_b.right

        jugador["rect"].y += jugador["vy"]
        jugador["en_suelo"] = False
        for (bx, by), d_b in mapa.items():
            r_b = pygame.Rect(bx, by, TAMANO_BLOQUE, TAMANO_BLOQUE)
            if jugador["rect"].colliderect(r_b):
                if jugador["vy"] > 0: 
                    jugador["rect"].bottom = r_b.top; jugador["vy"] = 0; jugador["en_suelo"] = True
                elif jugador["vy"] < 0: 
                    jugador["rect"].top = r_b.bottom; jugador["vy"] = 0

        # Físicas del Balón
        if balon["tipo_tiro"] == "parabola" and balon["vy"] > -2: 
            balon["vy"] += 1.2
        else: 
            balon["vy"] += 0.35
            
        balon["rect"].x += balon["vx"]
        balon["rect"].y += balon["vy"]
        balon["vx"] *= 0.98

        # Destrucción con el Tiro
        for (bx, by), d_b in list(mapa.items()):
            r_b = pygame.Rect(bx, by, TAMANO_BLOQUE, TAMANO_BLOQUE)
            if balon["rect"].colliderect(r_b):
                if balon["tipo_tiro"] == "tigre" and d_b["tipo"] != 5:
                    del mapa[(bx, by)]
                    crear_particulas(bx + 16, by + 16, COLOR_TIERRA)
                else:
                    if balon["vy"] > 0: 
                        balon["rect"].bottom = r_b.top; balon["vy"] = -balon["vy"] * 0.6
                    else: 
                        balon["vx"] = -balon["vx"] * 0.7

        if jugador["rect"].colliderect(balon["rect"]) and balon["tipo_tiro"] is None:
            balon["vx"] = jugador["vx"] * 1.2; balon["vy"] = -1.2

        # Ráfagas Ki destruyen bloques
        for kb in rafagas_ki[:]:
            kb["rect"].x += kb["vx"]
            for (bx, by), d_b in list(mapa.items()):
                if kb["rect"].colliderect(pygame.Rect(bx, by, TAMANO_BLOQUE, TAMANO_BLOQUE)):
                    if d_b["tipo"] != 5: 
                        del mapa[(bx, by)]
                    if kb in rafagas_ki: 
                        rafagas_ki.remove(kb)
                    break

    # CÁMARA
    camara_x += (jugador["rect"].centerx - ANCHO // 2 - camara_x) * 0.1
    camara_y += (jugador["rect"].centery - ALTO // 2 - camara_y) * 0.1

    # 3. RENDERIZADO
    pantalla.fill(COLOR_CIELO)

    # Dibujar Mapa
    for (bx, by), d_b in mapa.items():
        rx, ry = bx - camara_x, by - camara_y
        if -TAMANO_BLOQUE < rx < ANCHO and -TAMANO_BLOQUE < ry < ALTO:
            col = TIPOS_BLOQUES[d_b["tipo"]]["color"]
            pygame.draw.rect(pantalla, col, (rx, ry, TAMANO_BLOQUE, TAMANO_BLOQUE))
            if d_b["tipo"] == 1: 
                pygame.draw.rect(pantalla, COLOR_PASTO, (rx, ry, TAMANO_BLOQUE, 6))
            pygame.draw.rect(pantalla, (0, 0, 0), (rx, ry, TAMANO_BLOQUE, TAMANO_BLOQUE), 1)

    # Dibujar Balón y Jugador
    pygame.draw.ellipse(pantalla, (255, 255, 255), (balon["rect"].x - camara_x, balon["rect"].y - camara_y, 20, 20))
    
    px, py = jugador["rect"].x - camara_x, jugador["rect"].y - camara_y
    pygame.draw.rect(pantalla, COLOR_HEROE, (px, py, 28, 48))
    pelo_c = COLOR_PELO_SSJ if jugador["es_ssj"] else COLOR_PELO_BASE
    pygame.draw.polygon(pantalla, pelo_c, [(px - 2, py), (px + 10, py - 10), (px + 20, py), (px + 30, py - 8)])

    for kb in rafagas_ki:
        pygame.draw.ellipse(pantalla, kb["color"], (kb["rect"].x - camara_x, kb["rect"].y - camara_y, 16, 12))

    # UI / Interfaz
    pygame.draw.rect(pantalla, (40, 40, 40), (20, 20, 180, 16))
    pygame.draw.rect(pantalla, COLOR_PELO_SSJ if jugador["es_ssj"] else COLOR_KI, (20, 20, (jugador["ki"] / jugador["max_ki"]) * 180, 16))
    
    fuente = pygame.font.SysFont("Consolas", 16, bold=True)
    pantalla.blit(fuente.render(f"KI: {int(jugador['ki'])}%", True, (255, 255, 255)), (210, 18))
    b_nom = TIPOS_BLOQUES[jugador["bloque_sel"]]["nombre"]
    pantalla.blit(fuente.render(f"Bloque [1-5]: {b_nom}", True, TIPOS_BLOQUES[jugador["bloque_sel"]]["color"]), (20, 45))
    pantalla.blit(fuente.render("Presiona 'B' para invocar el balón", True, (200, 200, 200)), (20, 70))

    # Carga de Tiro (Barra flotante sobre el personaje)
    if cargando_tiro:
        bx_u, by_u = px - 20, py - 30
        pygame.draw.rect(pantalla, (30, 30, 30), (bx_u, by_u, 80, 10))
        pygame.draw.rect(pantalla, (46, 204, 113), (bx_u + 60, by_u, 16, 10))
        pygame.draw.rect(pantalla, COLOR_FUEGO, (bx_u, by_u, int((potencia_tiro / 100.0) * 80), 10))

    # Cinemática Anime
    if temp_cinematica > 0:
        temp_cinematica -= 1
        b_r = pygame.Rect(0, ALTO // 2 - 60, ANCHO, 120)
        pygame.draw.rect(pantalla, (10, 10, 20), b_r)
        pygame.draw.line(pantalla, datos_cinematica["color"], (0, b_r.top), (ANCHO, b_r.top), 4)
        pygame.draw.line(pantalla, datos_cinematica["color"], (0, b_r.bottom), (ANCHO, b_r.bottom), 4)
        
        f_cin = pygame.font.SysFont("Impact", 50)
        txt = f_cin.render(datos_cinematica["nombre"], True, datos_cinematica["color"])
        pantalla.blit(txt, (ANCHO // 2 - txt.get_width() // 2, ALTO // 2 - 30))

    pygame.display.flip()

pygame.quit()