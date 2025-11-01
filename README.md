#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define MAX_BLOCKS 10
#define MAX_HEIGHT 10

typedef struct {
    char tipo;
    int tamanho;
} Bloco;

Bloco pilha[MAX_BLOCKS];
int topo = -1;
int pontuacao = 0;

void desenharPilha() {
    printf("\n=== PILHA ATUAL ===\n");
    for (int i = topo; i >= 0; i--) {
        for (int j = 0; j < pilha[i].tamanho; j++) {
            printf("%c", pilha[i].tipo);
        }
        printf("\n");
    }
    if (topo == -1)
        printf("[vazio]\n");
    printf("====================\n");
}

int pilhaCheia() {
    return topo == MAX_HEIGHT - 1;
}

int pilhaVazia() {
    return topo == -1;
}

void empilhar(Bloco b) {
    if (pilhaCheia()) {
        printf("❌ A pilha está cheia! Game Over!\n");
        printf("Pontuação final: %d\n", pontuacao);
        exit(0);
    } else {
        topo++;
        pilha[topo] = b;
        pontuacao += b.tamanho * 10;
        printf("✅ Bloco empilhado com sucesso!\n");
    }
}

void removerBloco() {
    if (pilhaVazia()) {
        printf("⚠️ A pilha está vazia!\n");
    } else {
        printf("🧱 Removendo bloco do topo (%c)...\n", pilha[topo].tipo);
        topo--;
    }
}

Bloco gerarBloco() {
    Bloco b;
    int tipo = rand() % 3;
    switch (tipo) {
        case 0: b.tipo = '#'; break;
        case 1: b.tipo = '@'; break;
        case 2: b.tipo = '='; break;
    }
    b.tamanho = (rand() % 5) + 1; // de 1 a 5
    return b;
}

void menu() {
    printf("\n====== TETRIS STACK ======\n");
    printf("1. Gerar e empilhar bloco\n");
    printf("2. Remover bloco do topo\n");
    printf("3. Ver pilha\n");
    printf("4. Sair\n");
    printf("===========================\n");
    printf("Escolha uma opção: ");
}

int main() {
    srand(time(NULL));
    int opcao;

    printf("🎮 Bem-vindo ao TETRIS STACK!\n");
    printf("Objetivo: empilhar blocos o máximo possível sem deixar cair!\n");

    while (1) {
        menu();
        scanf("%d", &opcao);
        switch (opcao) {
            case 1: {
                Bloco novo = gerarBloco();
                printf("Novo bloco gerado: %c (tamanho %d)\n", novo.tipo, novo.tamanho);
                empilhar(novo);
                desenharPilha();
                break;
            }
            case 2:
                removerBloco();
                desenharPilha();
                break;
            case 3:
                desenharPilha();
                break;
            case 4:
                printf("👋 Saindo do jogo...\n");
                printf("Pontuação final: %d\n", pontuacao);
                return 0;
            default:
                printf("Opção inválida!\n");
        }
    }

    return 0;