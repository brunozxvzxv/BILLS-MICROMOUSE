## Maze Solving Logic & Flood-Fill Algorithm

This section documents the high-level decision-making software used by **Bills** to solve the maze grid within the **MMS (Micromouse Simulator)** environment.

###  Algorithm Strategy
* **Dynamic Flood-Fill Gradient:** Generates an array matrix where the target cells (the 4 center cells of an $8 \times 8$ or $16 \times 16$ grid) carry a distance weight of `0`. The values increase progressively toward the outer walls.
* **Look-Ahead Decision Matrix:** At each cell intersection, the robot scans its virtual wall distances (`wallFront()`, `wallLeft()`, `wallRight()`) and reads the neighboring matrix weights.
* **Gradient Descent Navigation:** The robot shifts its coordinate tracking vectors ($x, y$) and updates orientations (`NORTE`, `LESTE`, `SUL`, `OESTE`), always choosing the adjacent path containing the lowest mathematical value.

---

###  Simulation Control Code (`mms-logic.cpp`)


```cpp
//#define TAM16
#define TAM8

// Orientation vectors
#define NORTE 0
#define LESTE 1
#define SUL   2
#define OESTE 3

#ifdef TAM16
#define TAM 16
#endif

#ifdef TAM8
#define TAM 8
#endif

// Global positional tracking coordinates
int x = 0;
int y = 0;
int direcao = NORTE;
int countbeco = 0;

// Memory matrix representing distance values to the center
int mapa[TAM][TAM];


// ---------- GRADIENT MAP INITIALIZATION (4 CENTERS) ----------

void inicializaMapa() {
    for (int i = 0; i < TAM; i++) {
        for (int j = 0; j < TAM; j++) {

#ifdef TAM16
            // Calculate Manhattan distance metrics targeting the 4 center cells of a 16x16 grid
            mapa[i][j] = min(
                min(abs(i - 7) + abs(j - 7), abs(i - 7) + abs(j - 8)),
                min(abs(i - 8) + abs(j - 7), abs(i - 8) + abs(j - 8))
            );
#endif

#ifdef TAM8
            // Calculate Manhattan distance metrics targeting the 4 center cells of an 8x8 grid
            mapa[i][j] = min(
                min(abs(i - 3) + abs(j - 3), abs(i - 3) + abs(j - 4)),
                min(abs(i - 4) + abs(j - 3), abs(i - 4) + abs(j - 4))
            );
#endif

        }
    }
}


// ---------- CENTER BOUNDARY CHECK ----------

bool noCentro() {
#ifdef TAM16
    return ((x == 7 || x == 8) && (y == 7 || y == 8));
#endif
#ifdef TAM8
    return ((x == 3 || x == 4) && (y == 3 || y == 4));
#endif
}


// ---------- DEBUG LOGGING ----------

void printPosicao() {
    log("X:" + String(x) +
        " Y:" + String(y) +
        " Dir:" + String(direcao) +
        " Mapa:" + String(mapa[x][y]));
}


// ---------- NEIGHBOR VALUE FETCHING BY ORIENTATION ----------

int valorFrente() {
    if (direcao == NORTE && y + 1 < TAM) return mapa[x][y + 1];
    if (direcao == LESTE && x + 1 < TAM) return mapa[x + 1][y];
    if (direcao == SUL   && y - 1 >= 0 ) return mapa[x][y - 1];
    if (direcao == OESTE && x - 1 >= 0 ) return mapa[x - 1][y];
    return 999; // Boundary overflow protection
}

int valorDireita() {
    if (direcao == NORTE && x + 1 < TAM) return mapa[x + 1][y];
    if (direcao == LESTE && y - 1 >= 0 ) return mapa[x][y - 1];
    if (direcao == SUL   && x - 1 >= 0 ) return mapa[x - 1][y];
    if (direcao == OESTE && y + 1 < TAM) return mapa[x][y + 1];
    return 999;
}

int valorEsquerda() {
    if (direcao == NORTE && x - 1 >= 0 ) return mapa[x - 1][y];
    if (direcao == LESTE && y + 1 < TAM) return mapa[x][y + 1];
    if (direcao == SUL   && x + 1 < TAM) return mapa[x + 1][y];
    if (direcao == OESTE && y - 1 >= 0 ) return mapa[x][y - 1];
    return 999;
}


// ---------- MOTION PRIMITIVES EXECUTION ----------

void andaFrente() {
    if (moveForward()) {
        // Update grid positions according to current heading vector
        if (direcao == NORTE) y++;
        else if (direcao == LESTE) x++;
        else if (direcao == SUL) y--;
        else if (direcao == OESTE) x--;
    }
}

void viraDireita() {
    turnRight();
    direcao = (direcao + 1) % 4; // Shift orientation rightward
}

void viraEsquerda() {
    turnLeft();
    direcao = (direcao + 3) % 4; // Shift orientation leftward
}

void meiaVolta() {
    turnLeft();
    turnLeft();
    direcao = (direcao + 2) % 4; // Invert heading direction
}


// ---------- DEAD-END DETECTION ----------

bool travado() {
    return wallFront() && wallLeft() && wallRight();
}


// ---------- SETUP FUNCTION ----------

void setup() {
    Serial.begin(19200);
    log("System Ready!");

    inicializaMapa();
    setColor(x, y, 'R'); // Mark starting cell color on UI
}


// ---------- MAIN DECISION CONTROL LOOP ----------

void loop() {
    printPosicao();

    // Verification check for target goal reach
    if (noCentro()) {
        log("Goal reached at the center!");
        setColor(x, y, 'B'); // Mark target cell blue
        while (1);          // Halt program execution
    }

    // Dead-end recovery handler
    if (travado()) {
        log("Dead end encountered. Initiating turn back maneuver...");

        meiaVolta();
        andaFrente();
        setColor(x, y, 'Y'); // Highlight visited dead-end block in yellow

        delay(300);
        countbeco++;

        if (countbeco == 2)
            direcao = NORTE; // Force realignment escape clause

        return;
    }

    // Read distance map weights if path is free from walls
    int frente   = (!wallFront()) ? valorFrente()  : 999;
    int direita  = (!wallRight()) ? valorDireita() : 999;
    int esquerda = (!wallLeft())  ? valorEsquerda(): 999;

    // Follow path with lowest gradient cost value
    if (frente <= direita && frente <= esquerda) {
        andaFrente();
    }
    else if (direita < esquerda) {
        viraDireita();
        andaFrente();
    }
    else {
        viraEsquerda();
        andaFrente();
    }

    setColor(x, y, 'G'); // Mark traveled cell track in green
    delay(300);
}
