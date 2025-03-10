#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Definindo a estrutura para uma carta de Supertrunfo
struct CartaSupertrunfo {
    char pais[50];
    char codigo[10];
    char nome[50];
    long populacao;
    float area;
    float pib;
    int pontosTuristicos;
};

// Função para cadastrar uma carta
void cadastrarCarta(struct CartaSupertrunfo *carta) {
    printf("Digite o nome do pais: ");
    fgets(carta->pais, sizeof(carta->pais), stdin);
    carta->pais[strcspn(carta->pais, "\n")] = 0; // Remover o \n no final da string

    printf("Digite o código do pais: ");
    fgets(carta->codigo, sizeof(carta->codigo), stdin);
    carta->codigo[strcspn(carta->codigo, "\n")] = 0; // Remover o \n no final da string

    printf("Digite o nome da carta: ");
    fgets(carta->nome, sizeof(carta->nome), stdin);
    carta->nome[strcspn(carta->nome, "\n")] = 0; // Remover o \n no final da string

    printf("Digite a população do pais: ");
    scanf("%ld", &carta->populacao);

    printf("Digite a área do pais (em km²): ");
    scanf("%f", &carta->area);

    printf("Digite o PIB do pais (em trilhões de dólares): ");
    scanf("%f", &carta->pib);

    printf("Digite o número de pontos turísticos: ");
    scanf("%d", &carta->pontosTuristicos);

    // Limpeza do buffer de entrada
    while (getchar() != '\n');
}

// Função para exibir as informações de uma carta
void exibirCarta(struct CartaSupertrunfo carta) {
    printf("\n--- Carta do Supertrunfo ---\n");
    printf("País: %s\n", carta.pais);
    printf("Código: %s\n", carta.codigo);
    printf("Nome: %s\n", carta.nome);
    printf("População: %ld\n", carta.populacao);
    printf("Área: %.2f km²\n", carta.area);
    printf("PIB: $%.2f trilhões\n", carta.pib);
    printf("Pontos turísticos: %d\n", carta.pontosTuristicos);
    printf("\n");
}

int main() {
    int numCartas, i;
    
    // Solicitar o número de cartas que o usuário quer cadastrar
    printf("Digite o número de cartas que deseja cadastrar: ");
    scanf("%d", &numCartas);
    
    // Limpeza do buffer de entrada
    while (getchar() != '\n');
    
    struct CartaSupertrunfo cartas[numCartas];
    
    // Cadastrar as cartas
    for (i = 0; i < numCartas; i++) {
        printf("\nCadastro da carta %d\n", i + 1);
        cadastrarCarta(&cartas[i]);
    }

    // Exibir as cartas cadastradas
    for (i = 0; i < numCartas; i++) {
        exibirCarta(cartas[i]);
    }
    
    return 0;
}