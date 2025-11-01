/*
 freefire_sim.c
 Simulação em texto inspirada em Free Fire (versão "Free Fire - BattleSim")
 Autor: gerado por ChatGPT para Victor
 Compilar: gcc -std=c99 freefire_sim.c -o freefire_sim
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h>

#define MAX_NAME 32
#define MAX_PLAYERS 16
#define MAP_SIZE 100.0
#define MAX_ROUNDS 40
#define SAVE_FILE "freefire_save.dat"

typedef enum { WEAP_NONE=0, WEAP_PISTOL, WEAP_RIFLE, WEAP_SHOTGUN } WeaponType;

typedef struct {
    char name[MAX_NAME];
    int human; /* 1 se for o jogador humano, 0 bot */
    int alive;
    double x, y;
    int hp; /* 0..100 */
    int armor; /* 0..50 */
    WeaponType weapon;
    int medkits; /* quantidade */
    int kills;
} Player;

typedef struct {
    Player players[MAX_PLAYERS];
    int num_players;
    int round; /* rodada atual */
    double circle_x, circle_y;
    double circle_radius;
    int alive_count;
} GameState;

/* armas: dano base, range, accuracy (0-100) */
typedef struct { int base_dmg; double range; int accuracy; char *name; } WeaponInfo;
WeaponInfo weapon_info[] = {
    {0, 0.0, 0, "None"},
    {22, 30.0, 60, "Pistol"},
    {35, 50.0, 70, "Rifle"},
    {45, 12.0, 55, "Shotgun"}
};

/* util */
void read_line(char *buf, int n) {
    if (fgets(buf, n, stdin)) {
        size_t L = strlen(buf);
        if (L && buf[L-1]=='\n') buf[L-1]='\0';
    } else {
        buf[0] = '\0';
    }
}

double dist(double x1, double y1, double x2, double y2) {
    double dx = x1-x2, dy = y1-y2;
    return sqrt(dx*dx + dy*dy);
}

int weapon_is_valid(WeaponType w) {
    return (w>=WEAP_PISTOL && w<=WEAP_SHOTGUN);
}

const char* weapon_name(WeaponType w) {
    if (!weapon_is_valid(w)) return "None";
    return weapon_info[w].name;
}

/* init: pos random no mapa */
void random_pos(double *x, double *y) {
    *x = (double)rand()/(RAND_MAX+1.0) * MAP_SIZE;
    *y = (double)rand()/(RAND_MAX+1.0) * MAP_SIZE;
}

/* Cria novo estado de jogo */
void init_game(GameState g, int num_players, const char human_name) {
    memset(g,0,sizeof(GameState));
    g->num_players = num_players;
    g->round = 1;
    g->circle_x = MAP_SIZE/2.0;
    g->circle_y = MAP_SIZE/2.0;
    g->circle_radius = MAP_SIZE * 0.6;
    g->alive_count = num_players;
    for (int i=0;i<num_players;i++) {
        Player *p = &g->players[i];
        snprintf(p->name, MAX_NAME, "Bot%02d", i+1);
        p->human = 0;
        p->alive = 1;
        random_pos(&p->x, &p->y);
        p->hp = 100;
        p->armor = (rand()%3)? (rand()%31) : 0; /* alguns bots com colete */
        p->weapon = (rand()%3)+1; /* arma aleatória */
        p->medkits = rand()%2;
        p->kills = 0;
    }
    /* substitui o primeiro player por humano (se pedido) */
    if (human_name && strlen(human_name)>0) {
        Player *h = &g->players[0];
        snprintf(h->name, MAX_NAME, "%s", human_name);
        h->human = 1;
        h->weapon = WEAP_NONE;
        h->armor = 0;
        h->medkits = 1;
        random_pos(&h->x, &h->y);
        h->hp = 100;
    }
}

/* Mostrar HUD simples */
void print_hud(GameState *g) {
    printf("\n=== ROUND %d | Players vivos: %d | Zona: centro(%.1f,%.1f) raio %.1f ===\n",
           g->round, g->alive_count, g->circle_x, g->circle_y, g->circle_radius);
}

/* Mostrar status do jogador humano */
void print_player_status(Player *p) {
    printf("\n--- Você (%s) ---\n", p->name);
    printf("HP: %d | Armor: %d | Arma: %s | Medkits: %d | Posição: (%.1f,%.1f) | Kills: %d\n",
           p->hp, p->armor, weapon_name(p->weapon), p->medkits, p->x, p->y, p->kills);
}

/* Buscar jogador vivo por índice (salta se morto) */
int alive_players(GameState *g) {
    int c=0;
    for (int i=0;i<g->num_players;i++) if (g->players[i].alive) c++;
    return c;
}

/* Mover jogador (input) */
void move_player(Player *p) {
    char buf[64];
    double nx, ny;
    printf("Mover: digite novo X e Y (0..%.0f), ex: 50 30 > ", MAP_SIZE);
    read_line(buf, sizeof(buf));
    if (sscanf(buf, "%lf %lf", &nx, &ny) == 2) {
        if (nx < 0) nx = 0; if (ny < 0) ny = 0;
        if (nx > MAP_SIZE) nx = MAP_SIZE; if (ny > MAP_SIZE) ny = MAP_SIZE;
        p->x = nx; p->y = ny;
        printf("Movido para (%.1f,%.1f)\n", p->x, p->y);
    } else {
        printf("Entrada inválida. Movimento cancelado.\n");
    }
}

/* Loot simples: chance de achar arma melhor ou medkit */
void loot_action(Player *p) {
    int roll = rand()%100;
    if (roll < 40) {
        /* medkit */
        p->medkits += 1;
        printf("Você encontrou um MedKit! Medkits agora: %d\n", p->medkits);
    } else {
        /* arma chance de melhorar */
        WeaponType w = (rand()%3)+1;
        if (weapon_info[w].base_dmg > weapon_info[p->weapon].base_dmg) {
            p->weapon = w;
            printf("Você achou uma arma melhor: %s\n", weapon_name(w));
        } else {
            printf("Você encontrou %s, mas já tem arma igual ou melhor.\n", weapon_name(w));
        }
    }
}

/* Tenta usar medkit */
void use_medkit(Player *p) {
    if (p->medkits <= 0) {
        printf("Sem medkits.\n"); return;
    }
    p->medkits--;
    int heal = 35 + rand()%16; /* 35-50 */
    p->hp += heal;
    if (p->hp > 100) p->hp = 100;
    printf("Você usou um MedKit e recuperou %d HP. HP agora: %d\n", heal, p->hp);
}

/* Jogada de tiro: atacante tenta acertar alvo */
void shoot(Player *att, Player *def, GameState *g) {
    if (!weapon_is_valid(att->weapon)) {
        printf("%s tentou atirar sem arma!\n", att->name);
        return;
    }
    double d = dist(att->x, att->y, def->x, def->y);
    WeaponInfo wi = weapon_info[att->weapon];
    if (d > wi.range) {
        /* fora de alcance */
        printf("%s tentou atirar em %s, mas estava fora do alcance (dist %.1f > %.1f).\n", att->name, def->name, d, wi.range);
        return;
    }
    /* chance de acertar = accuracy - (dist/range)*30 +/- 10 random */
    double dist_factor = (d / wi.range) * 30.0;
    int base_acc = wi.accuracy - (int)dist_factor;
    int acc_roll = base_acc + (rand()%21 - 10); /* +/-10 randomness */
    if (acc_roll < 5) acc_roll = 5;
    if (acc_roll > 95) acc_roll = 95;
    int hit = (rand()%100) < acc_roll;
    if (!hit) {
        if (att->human) printf("Você atirou e errou %s (chance %d%%).\n", def->name, acc_roll);
        else if (def->human) printf("%s atirou em você e errou! (chance %d%%)\n", att->name, acc_roll);
        return;
    }
    /* dano reduzido pela armor: effective armor mitigates up to 50% */
    int dmg = wi.base_dmg + (rand()%(wi.base_dmg/3 + 1)); /* variação */
    int mitigation = (int)(def->armor * 0.6); /* parte efetiva */
    int final = dmg - (mitigation / 10);
    if (final < 1) final = 1;
    def->hp -= final;
    if (att->human) printf("Você acertou %s e causou %d de dano. HP restante: %d\n", def->name, final, def->hp);
    else if (def->human) printf("%s acertou você e causou %d de dano. Seu HP: %d\n", att->name, final, def->hp);
    else {
        /* opcional mensagem de bots atacando bots -- omitido para poluir menos */
    }
    if (def->hp <= 0) {
        def->alive = 0;
        def->hp = 0;
        att->kills++;
        g->alive_count = alive_players(g);
        if (att->human) printf("Você eliminou %s!\n", def->name);
        else if (def->human) printf("Você foi eliminado por %s!\n", att->name);
        else {
            /* bot vs bot killed */
        }
    }
}

/* Zona segura: aplica dano a quem está fora */
void apply_zone_damage(GameState *g) {
    for (int i=0;i<g->num_players;i++) {
        Player *p = &g->players[i];
        if (!p->alive) continue;
        double d = dist(p->x, p->y, g->circle_x, g->circle_y);
        if (d > g->circle_radius) {
            int dmg = 8 + rand()%8; /* 8-15 por rodada fora */
            p->hp -= dmg;
            if (p->hp <= 0) {
                p->hp = 0; p->alive = 0;
                printf("%s foi eliminado por estar fora da zona segura!\n", p->name);
            } else {
                if (p->human) printf("Você levou %d de dano por estar fora da zona segura. HP: %d\n", dmg, p->hp);
            }
        }
    }
    g->alive_count = alive_players(g);
}

/* Bots AI decisão simples */
void bot_action(Player *bot, GameState *g) {
    if (!bot->alive) return;
    /* Se pouca vida e tem medkit -> usar */
    if (bot->hp < 40 && bot->medkits > 0 && rand()%100 < 70) {
        bot->medkits--; bot->hp += 35 + rand()%16;
        if (bot->hp > 100) bot->hp = 100;
        return;
    }
    /* Se sem arma -> loot (chance de procurar) */
    if (!weapon_is_valid(bot->weapon) && rand()%100 < 70) {
        bot->weapon = (rand()%3)+1;
        return;
    }
    /* Encontrar inimigo próximo */
    int target = -1;
    double bestd = 1e9;
    for (int j=0;j<g->num_players;j++) {
        if (&g->players[j] == bot) continue;
        if (!g->players[j].alive) continue;
        double d = dist(bot->x, bot->y, g->players[j].x, g->players[j].y);
        if (d < bestd) { bestd = d; target = j; }
    }
    if (target != -1 && bestd < weapon_info[bot->weapon].range && rand()%100 < 75) {
        /* atira */
        shoot(bot, &g->players[target], g);
        return;
    }
    /* senão, aleatoriamente mover em direção ao centro ou loot */
    if (rand()%100 < 50) {
        /* mover 10 unidades aleatórias */
        double ang = (double)rand() / RAND_MAX * 2.0 * M_PI;
        double step = 5.0 + (rand()%8);
        bot->x += cos(ang) * step;
        bot->y += sin(ang) * step;
        if (bot->x < 0) bot->x = 0; if (bot->x > MAP_SIZE) bot->x = MAP_SIZE;
        if (bot->y < 0) bot->y = 0; if (bot->y > MAP_SIZE) bot->y = MAP_SIZE;
    } else {
        /* loot chance: increase medkits or weapon */
        if (rand()%100 < 60) {
            if (rand()%100 < 50) bot->medkits++;
            else bot->weapon = (rand()%3)+1;
        }
    }
}

/* Exibe leaderboard e players vivos */
void print_players(GameState *g) {
    printf("\n--- Players ---\n");
    for (int i=0;i<g->num_players;i++) {
        Player *p = &g->players[i];
        printf("%2d) %s %s | HP:%3d ARM:%2d W:%s K:%d Pos:(%.1f,%.1f)\n",
               i+1, p->name, p->alive ? "(Vivo)":"(Morto)", p->hp, p->armor, weapon_name(p->weapon), p->kills, p->x, p->y);
    }
}

/* Salvar e carregar */
void save_game(GameState *g) {
    FILE *f = fopen(SAVE_FILE, "wb");
    if (!f) { printf("Erro ao abrir arquivo para salvar.\n"); return; }
    fwrite(g, sizeof(GameState), 1, f);
    fclose(f);
    printf("Jogo salvo em '%s'.\n", SAVE_FILE);
}

int load_game(GameState *g) {
    FILE *f = fopen(SAVE_FILE, "rb");
    if (!f) { printf("Arquivo de save não encontrado.\n"); return 0; }
    fread(g, sizeof(GameState), 1, f);
    fclose(f);
    printf("Jogo carregado.\n");
    return 1;
}

/* Encolhe a zona: move o centro levemente e reduz o raio */
void shrink_zone(GameState *g) {
    /* move o centro um pouco para uma direção aleatória e reduz raio */
    double ang = (double)rand()/RAND_MAX * 2.0 * M_PI;
    double step = (double)(rand()%6) - 3.0; /* -3..+2 */
    g->circle_x += cos(ang) * step;
    g->circle_y += sin(ang) * step;
    if (g->circle_x < 10) g->circle_x = 10;
    if (g->circle_x > MAP_SIZE-10) g->circle_x = MAP_SIZE-10;
    if (g->circle_y < 10) g->circle_y = 10;
    if (g->circle_y > MAP_SIZE-10) g->circle_y = MAP_SIZE-10;
    g->circle_radius = 0.85; / reduz 15% */
    if (g->circle_radius < 6.0) g->circle_radius = 6.0;
}

/* Loop principal de jogo */
void game_loop(GameState *g) {
    char cmd[128];
    while (g->alive_count > 1 && g->round <= MAX_ROUNDS) {
        print_hud(g);
        /* localizar jogador humano */
        int human_idx = -1;
        for (int i=0;i<g->num_players;i++) if (g->players[i].human) human_idx = i;
        if (human_idx >= 0 && g->players[human_idx].alive) {
            Player *me = &g->players[human_idx];
            print_player_status(me);
            printf("\nAções: move | loot | use | shoot <target_index> | status | players | salvar | carregar | passar\n");
            printf("Digite comando: ");
            read_line(cmd, sizeof(cmd));
            if (strncmp(cmd, "move", 4) == 0) {
                move_player(me);
            } else if (strncmp(cmd, "loot", 4) == 0) {
                loot_action(me);
            } else if (strncmp(cmd, "use", 3) == 0) {
                use_medkit(me);
            } else if (strncmp(cmd, "shoot", 5) == 0) {
                int tid = -1;
                if (sscanf(cmd+5, "%d", &tid) == 1) {
                    tid -= 1;
                    if (tid >= 0 && tid < g->num_players && g->players[tid].alive) {
                        shoot(me, &g->players[tid], g);
                    } else printf("Alvo inválido.\n");
                } else printf("Use: shoot <num_do_alvo>\n");
            } else if (strncmp(cmd, "status", 6) == 0) {
                /* só reimprime */
            } else if (strncmp(cmd, "players", 7) == 0) {
                print_players(g);
            } else if (strncmp(cmd, "salvar", 6) == 0) {
                save_game(g);
            } else if (strncmp(cmd, "carregar", 8) == 0) {
                if (load_game(g)) {
                    /* reconta vivos */
                    g->alive_count = alive_players(g);
                }
            } else if (strncmp(cmd, "passar", 6) == 0 || strlen(cmd)==0) {
                /* nada, passa vez */
            } else if (strncmp(cmd, "quit", 4) == 0) {
                printf("Você escolheu sair do jogo. Salvando e encerrando...\n");
                save_game(g);
                return;
            } else {
                printf("Comando desconhecido. Use 'move', 'loot', 'use', 'shoot <n>', 'players', 'salvar', 'carregar', 'passar'.\n");
            }
        } else {
            if (human_idx >= 0) {
                printf("Você está morto! Esperando fim da partida.\n");
            } else {
                printf("Sem jogador humano detectado (rodando só bots).\n");
            }
        }

        /* Bots agem */
        for (int i=0;i<g->num_players;i++) {
            Player *p = &g->players[i];
            if (!p->alive) continue;
            if (p->human) continue;
            bot_action(p, g);
        }

        /* aplicar danos de zona */
        apply_zone_damage(g);

        /* encolher zona a cada 3 rodadas para manter ritmo */
        if (g->round % 3 == 0) {
            shrink_zone(g);
            printf("\n-- A zona segura encolheu! --\n");
        }

        /* pequenas mensagens de progresso */
        if (g->round % 5 == 0) {
            printf("Round %d — jogadores restantes: %d\n", g->round, g->alive_count);
        }

        /* checar vitórias parciais */
        if (g->alive_count <= 1) break;

        g->round++;
    }

    /* fim de partida */
    printf("\n=== FIM DE PARTIDA ===\n");
    /* achar vencedor */
    int winner = -1;
    for (int i=0;i<g->num_players;i++) if (g->players[i].alive) winner = i;
    if (winner != -1) {
        printf("Vencedor: %s (Kills: %d)\n", g->players[winner].name, g->players[winner].kills);
        if (g->players[winner].human) printf("Parabéns! Você venceu a partida.\n");
    } else {
        printf("Nenhum vencedor (todos eliminados).\n");
    }
    printf("\nPlacar final:\n");
    print_players(g);
}

/* menu inicial */
int main(void) {
    srand((unsigned int)time(NULL));
    GameState g;
    memset(&g, 0, sizeof(g));

    printf("=== Free Fire - BattleSim (Texto) ===\n");
    printf("1) Novo jogo\n2) Carregar jogo salvo\nEscolha: ");
    char buf[64];
    read_line(buf, sizeof(buf));
    if (buf[0] == '2') {
        if (!load_game(&g)) {
            printf("Iniciando novo jogo por falha no load.\n");
            goto newgame;
        } else {
            printf("Continuando jogo salvo...\n");
            game_loop(&g);
            return 0;
        }
    }

newgame:
    int num;
    char name[MAX_NAME];
    printf("Nome do jogador (ou deixe vazio para só bots): ");
    read_line(name, sizeof(name));
    printf("Número total de jogadores (2-%d): ", MAX_PLAYERS);
    read_line(buf, sizeof(buf));
    num = atoi(buf);
    if (num < 2) num = 8;
    if (num > MAX_PLAYERS) num = MAX_PLAYERS;
    init_game(&g, num, name);
    printf("Jogo iniciado com %d jogadores.\n", g.num_players);
    if (strlen(name)>0) printf("Você joga como: %s\n", name);
    printf("Comandos: move, loot, use, shoot <num>, players, salvar, carregar, passar\n");
    game_loop(&g);
    return 0;
}