# cpp
/*******************************************************************************************
*
* Raylib - Classic Game: Tetris (C++ Refactored Version)
* 이 코드는 C++ 클래스 구조와 객체지향 방식을 사용하여 리팩토링되었습니다.
*
********************************************************************************************/

#include "raylib.h"
#include <vector>

//----------------------------------------------------------------------------------
// 상수 정의
//----------------------------------------------------------------------------------
const int SQUARE_SIZE = 20;
const int GRID_HORIZONTAL_SIZE = 12;
const int GRID_VERTICAL_SIZE = 20;
const int LATERAL_SPEED = 10;
const int TURNING_SPEED = 12;
const int FAST_FALL_AWAIT_COUNTER = 30;
const int FADING_TIME = 33;

//----------------------------------------------------------------------------------
// 타입 정의 (C++ enum class)
//----------------------------------------------------------------------------------
enum class SquareState { EMPTY, MOVING, FULL, BLOCK, FADING };

//----------------------------------------------------------------------------------
// 테트리스 게임 클래스
//----------------------------------------------------------------------------------
class TetrisGame {
private:
    SquareState grid[GRID_HORIZONTAL_SIZE][GRID_VERTICAL_SIZE];
    SquareState piece[4][4];
    SquareState incomingPiece[4][4];

    int piecePositionX = 0;
    int piecePositionY = 0;
    
    bool gameOver = false;
    bool pause = false;
    bool beginPlay = true;
    bool pieceActive = false;
    bool detection = false;
    bool lineToDelete = false;

    int lines = 0;
    int gravitySpeed = 30;
    int gravityMovementCounter = 0;
    int lateralMovementCounter = 0;
    int turnMovementCounter = 0;
    int fastFallMovementCounter = 0;
    int fadeLineCounter = 0;

    Color fadingColor = GRAY;

public:
    TetrisGame() { Init(); }

    // 게임 초기화
    void Init() {
        lines = 0; gravitySpeed = 30;
        gameOver = false; pause = false; beginPlay = true;
        pieceActive = false; detection = false; lineToDelete = false;
        
        for (int i = 0; i < GRID_HORIZONTAL_SIZE; i++) {
            for (int j = 0; j < GRID_VERTICAL_SIZE; j++) {
                if ((j == GRID_VERTICAL_SIZE - 1) || (i == 0) || (i == GRID_HORIZONTAL_SIZE - 1)) 
                    grid[i][j] = SquareState::BLOCK;
                else 
                    grid[i][j] = SquareState::EMPTY;
            }
        }
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) incomingPiece[i][j] = SquareState::EMPTY;
        }
    }

    // 매 프레임 업데이트
    void Update() {
        if (gameOver) {
            if (IsKeyPressed(KEY_ENTER)) Init();
            return;
        }

        if (IsKeyPressed('P')) pause = !pause;
        if (pause) return;

        if (!lineToDelete) {
            if (!pieceActive) {
                pieceActive = CreatePiece();
                fastFallMovementCounter = 0;
            } else {
                HandleInput();
                HandleGravity();
            }
            CheckGameOver();
        } else {
            HandleLineAnimation();
        }
    }

    // 화면 그리기
    void Draw() {
        BeginDrawing();
        ClearBackground(RAYWHITE);

        if (!gameOver) {
            Vector2 offset = { (float)GetScreenWidth()/2 - 120, (float)GetScreenHeight()/2 - 180 };

            // 보드 그리드 출력
            for (int j = 0; j < GRID_VERTICAL_SIZE; j++) {
                for (int i = 0; i < GRID_HORIZONTAL_SIZE; i++) {
                    DrawSquare(grid[i][j], offset.x + i * SQUARE_SIZE, offset.y + j * SQUARE_SIZE);
                }
            }

            // UI 정보 출력
            DrawText("NEXT PIECE:", offset.x + 260, offset.y, 15, DARKGRAY);
            for (int j = 0; j < 4; j++) {
                for (int i = 0; i < 4; i++) {
                    if (incomingPiece[i][j] == SquareState::MOVING)
                        DrawRectangle(offset.x + 260 + i*SQUARE_SIZE, offset.y + 30 + j*SQUARE_SIZE, SQUARE_SIZE, SQUARE_SIZE, GRAY);
                }
            }

            DrawText(TextFormat("LINES: %04i", lines), offset.x + 260, offset.y + 140, 20, MAROON);
            if (pause) DrawText("PAUSED", GetScreenWidth()/2 - 60, GetScreenHeight()/2 - 20, 30, DARKGRAY);
        } else {
            DrawText("GAME OVER", GetScreenWidth()/2 - 80, GetScreenHeight()/2 - 40, 30, MAROON);
            DrawText("PRESS [ENTER] TO RESTART", GetScreenWidth()/2 - 130, GetScreenHeight()/2 + 10, 20, DARKGRAY);
        }
        EndDrawing();
    }

private:
    void DrawSquare(SquareState state, int x, int y) {
        switch(state) {
            case SquareState::EMPTY: DrawRectangleLines(x, y, SQUARE_SIZE, SQUARE_SIZE, LIGHTGRAY); break;
            case SquareState::FULL: DrawRectangle(x, y, SQUARE_SIZE, SQUARE_SIZE, GRAY); break;
            case SquareState::MOVING: DrawRectangle(x, y, SQUARE_SIZE, SQUARE_SIZE, DARKGRAY); break;
            case SquareState::BLOCK: DrawRectangle(x, y, SQUARE_SIZE, SQUARE_SIZE, LIGHTGRAY); break;
            case SquareState::FADING: DrawRectangle(x, y, SQUARE_SIZE, SQUARE_SIZE, fadingColor); break;
        }
    }

    bool CreatePiece() {
        piecePositionX = (GRID_HORIZONTAL_SIZE - 4) / 2;
        piecePositionY = 0;

        if (beginPlay) { GetRandomPiece(incomingPiece); beginPlay = false; }

        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) piece[i][j] = incomingPiece[i][j];
        }

        GetRandomPiece(incomingPiece);

        // 생성 직후 충돌 체크 (게임 오버 판정용)
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                if (piece[i][j] == SquareState::MOVING) {
                    if (grid[piecePositionX + i][piecePositionY + j] == SquareState::FULL) return false;
                    grid[piecePositionX + i][piecePositionY + j] = SquareState::MOVING;
                }
            }
        }
        return true;
    }

    void GetRandomPiece(SquareState target[4][4]) {
        for (int i = 0; i < 4; i++) for (int j = 0; j < 4; j++) target[i][j] = SquareState::EMPTY;
        int rnd = GetRandomValue(0, 6);
        switch(rnd) {
            case 0: target[1][1]=target[2][1]=target[1][2]=target[2][2]=SquareState::MOVING; break; // Cube
            case 1: target[1][0]=target[1][1]=target[1][2]=target[2][2]=SquareState::MOVING; break; // L
            case 2: target[1][2]=target[2][0]=target[2][1]=target[2][2]=SquareState::MOVING; break; // L-inv
            case 3: target[0][1]=target[1][1]=target[2][1]=target[3][1]=SquareState::MOVING; break; // Line
            case 4: target[1][0]=target[1][1]=target[1][2]=target[2][1]=SquareState::MOVING; break; // T
            case 5: target[1][1]=target[2][1]=target[2][2]=target[3][2]=SquareState::MOVING; break; // S
            case 6: target[1][2]=target[2][2]=target[2][1]=target[3][1]=SquareState::MOVING; break; // S-inv
        }
    }

    void HandleInput() {
        lateralMovementCounter++;
        if (IsKeyPressed(KEY_LEFT) || IsKeyPressed(KEY_RIGHT)) lateralMovementCounter = LATERAL_SPEED;
        if (lateralMovementCounter >= LATERAL_SPEED) {
            ResolveLateralMovement();
            lateralMovementCounter = 0;
        }

        // 회전 조작
        if (IsKeyPressed(KEY_UP)) {
            ResolveTurnMovement();
        }
    }

    void HandleGravity() {
        gravityMovementCounter++;
        fastFallMovementCounter++;

        if (IsKeyDown(KEY_DOWN) && (fastFallMovementCounter >= FAST_FALL_AWAIT_COUNTER)) gravityMovementCounter += gravitySpeed;

        if (gravityMovementCounter >= gravitySpeed) {
            CheckDetection();
            if (detection) {
                LockPiece();
                CheckCompletion();
            } else {
                MovePieceDown();
            }
            gravityMovementCounter = 0;
        }
    }

    bool ResolveTurnMovement() {
        SquareState aux[4][4];
        bool collision = false;

        // 1. 임시 배열에 90도 회전 상태 계산
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                aux[j][3 - i] = piece[i][j];
            }
        }

        // 2. 충돌 체크
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                if (aux[i][j] == SquareState::MOVING) {
                    int nx = piecePositionX + i;
                    int ny = piecePositionY + j;
                    if (nx <= 0 || nx >= GRID_HORIZONTAL_SIZE - 1 || ny >= GRID_VERTICAL_SIZE - 1 || 
                        grid[nx][ny] == SquareState::FULL) {
                        collision = true;
                        break;
                    }
                }
            }
        }

        // 3. 충돌 없으면 적용
        if (!collision) {
            for (int i = 0; i < 4; i++) {
                for (int j = 0; j < 4; j++) {
                    if (piece[i][j] == SquareState::MOVING) grid[piecePositionX + i][piecePositionY + j] = SquareState::EMPTY;
                }
            }
            for (int i = 0; i < 4; i++) {
                for (int j = 0; j < 4; j++) {
                    piece[i][j] = aux[i][j];
                    if (piece[i][j] == SquareState::MOVING) grid[piecePositionX + i][piecePositionY + j] = SquareState::MOVING;
                }
            }
            return true;
        }
        return false;
    }

    void MovePieceDown() {
        for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--) {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
                if (grid[i][j] == SquareState::MOVING) {
                    grid[i][j+1] = SquareState::MOVING;
                    grid[i][j] = SquareState::EMPTY;
                }
            }
        }
        piecePositionY++;
    }

    void LockPiece() {
        for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
            for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++) {
                if (grid[i][j] == SquareState::MOVING) grid[i][j] = SquareState::FULL;
            }
        }
        detection = false;
        pieceActive = false;
    }

    void CheckDetection() {
        for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++) {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
                if (grid[i][j] == SquareState::MOVING) {
                    if (grid[i][j+1] == SquareState::FULL || grid[i][j+1] == SquareState::BLOCK) {
                        detection = true; return;
                    }
                }
            }
        }
    }

    void CheckCompletion() {
        for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++) {
            int count = 0;
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
                if (grid[i][j] == SquareState::FULL) count++;
            }
            if (count == GRID_HORIZONTAL_SIZE - 2) {
                lineToDelete = true;
                for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) grid[i][j] = SquareState::FADING;
            }
        }
    }

    void HandleLineAnimation() {
        fadeLineCounter++;
        fadingColor = (fadeLineCounter % 8 < 4) ? MAROON : GRAY;
        if (fadeLineCounter >= FADING_TIME) {
            DeleteLines();
            fadeLineCounter = 0;
            lineToDelete = false;
        }
    }

    void DeleteLines() {
        for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++) {
            if (grid[1][j] == SquareState::FADING) {
                for (int row = j; row > 0; row--) {
                    for (int col = 1; col < GRID_HORIZONTAL_SIZE - 1; col++) grid[col][row] = grid[col][row-1];
                }
                lines++;
            }
        }
    }

    void ResolveLateralMovement() {
        int dir = 0;
        if (IsKeyDown(KEY_LEFT)) dir = -1;
        else if (IsKeyDown(KEY_RIGHT)) dir = 1;
        if (dir == 0) return;

        bool collision = false;
        for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++) {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
                if (grid[i][j] == SquareState::MOVING) {
                    if (grid[i+dir][j] == SquareState::FULL || grid[i+dir][j] == SquareState::BLOCK) collision = true;
                }
            }
        }

        if (!collision) {
            if (dir == -1) {
                for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) 
                    for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++)
                        if (grid[i][j] == SquareState::MOVING) { grid[i-1][j] = SquareState::MOVING; grid[i][j] = SquareState::EMPTY; }
            } else {
                for (int i = GRID_HORIZONTAL_SIZE - 2; i >= 1; i--)
                    for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++)
                        if (grid[i][j] == SquareState::MOVING) { grid[i+1][j] = SquareState::MOVING; grid[i][j] = SquareState::EMPTY; }
            }
            piecePositionX += dir;
        }
    }

    void CheckGameOver() {
        for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++) {
            if (grid[i][0] == SquareState::FULL) gameOver = true;
        }
    }
};

//------------------------------------------------------------------------------------
// Main
//------------------------------------------------------------------------------------
int main() {
    InitWindow(800, 450, "C++ Raylib Tetris - Final Version");
    SetTargetFPS(60);

    TetrisGame game;

    while (!WindowShouldClose()) {
        game.Update();
        game.Draw();
    }

    CloseWindow();
    return 0;
}
