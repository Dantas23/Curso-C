# Funções em C++

## Introdução às Funções

As funções são blocos de código reutilizáveis que executam tarefas específicas. Elas são fundamentais para organizar o código, evitar repetições e tornar o programa mais modular e fácil de manter. Em C++, as funções são particularmente poderosas devido à sua flexibilidade e recursos avançados como sobrecarga, funções inline e templates.

Para o desenvolvimento de cheats para jogos, as funções são essenciais, pois permitem encapsular operações complexas (como cálculos de aimbot, detecção de entidades ou manipulação de memória) em blocos de código organizados e reutilizáveis. Isso torna o código do cheat mais limpo, mais fácil de depurar e mais fácil de expandir com novas funcionalidades.

Vamos explorar os conceitos fundamentais de funções em C++ e entender como elas podem ser aplicadas no desenvolvimento de cheats para jogos.

## Declaração e Definição de Funções

Uma função em C++ consiste em duas partes principais: a declaração (ou protótipo) e a definição.

### Declaração de Função

A declaração de uma função informa ao compilador sobre a existência da função, seu nome, tipo de retorno e parâmetros. Ela termina com um ponto e vírgula e geralmente é colocada em arquivos de cabeçalho (.h):

```cpp
tipo_retorno nome_funcao(tipo_parametro1 parametro1, tipo_parametro2 parametro2, ...);
```

Exemplo:
```cpp
// Declaração de função
float calcular_distancia(float x1, float y1, float x2, float y2);
void ativar_aimbot(bool ativar);
int encontrar_jogador_mais_proximo(int* lista_jogadores, int num_jogadores);
```

### Definição de Função

A definição de uma função contém o código que será executado quando a função for chamada. Ela inclui o corpo da função entre chaves:

```cpp
tipo_retorno nome_funcao(tipo_parametro1 parametro1, tipo_parametro2 parametro2, ...) {
    // Corpo da função
    // Código a ser executado
    return valor;  // Se o tipo de retorno não for void
}
```

Exemplo:
```cpp
// Definição de função
float calcular_distancia(float x1, float y1, float x2, float y2) {
    float dx = x2 - x1;
    float dy = y2 - y1;
    return sqrt(dx*dx + dy*dy);
}

void ativar_aimbot(bool ativar) {
    if (ativar) {
        std::cout << "Aimbot ativado!" << std::endl;
        // Código para ativar o aimbot
    } else {
        std::cout << "Aimbot desativado." << std::endl;
        // Código para desativar o aimbot
    }
}
```

## Parâmetros e Argumentos

Os parâmetros são as variáveis listadas na declaração da função, enquanto os argumentos são os valores reais passados para a função quando ela é chamada.

### Passagem por Valor

Por padrão, os parâmetros são passados por valor, o que significa que a função recebe uma cópia do valor original. Alterações no parâmetro dentro da função não afetam o argumento original:

```cpp
void incrementar(int numero) {
    numero++;  // Modifica apenas a cópia local
    std::cout << "Dentro da função: " << numero << std::endl;
}

int main() {
    int valor = 5;
    incrementar(valor);
    std::cout << "Após a chamada: " << valor << std::endl;  // Ainda é 5
    return 0;
}
```

### Passagem por Referência

Para permitir que uma função modifique o valor original, podemos passar parâmetros por referência usando o operador `&`:

```cpp
void incrementar_ref(int& numero) {
    numero++;  // Modifica o valor original
    std::cout << "Dentro da função: " << numero << std::endl;
}

int main() {
    int valor = 5;
    incrementar_ref(valor);
    std::cout << "Após a chamada: " << valor << std::endl;  // Agora é 6
    return 0;
}
```

A passagem por referência é particularmente útil quando:
- Precisamos modificar o valor original
- Queremos evitar a cópia de objetos grandes (por eficiência)

### Passagem por Ponteiro

Outra forma de permitir que uma função modifique o valor original é passar um ponteiro:

```cpp
void incrementar_ptr(int* numero) {
    (*numero)++;  // Modifica o valor apontado
    std::cout << "Dentro da função: " << *numero << std::endl;
}

int main() {
    int valor = 5;
    incrementar_ptr(&valor);  // Passa o endereço de valor
    std::cout << "Após a chamada: " << valor << std::endl;  // Agora é 6
    return 0;
}
```

A passagem por ponteiro é comum em C++ quando:
- Trabalhamos com APIs de baixo nível
- Precisamos indicar valores opcionais (ponteiro pode ser nullptr)
- Manipulamos memória diretamente (como em cheats)

### Parâmetros Constantes

Para garantir que uma função não modifique um parâmetro, podemos usar a palavra-chave `const`:

```cpp
// Função que recebe uma referência constante
float calcular_media(const std::vector<int>& valores) {
    int soma = 0;
    for (int valor : valores) {
        soma += valor;
    }
    return static_cast<float>(soma) / valores.size();
}
```

Usar `const` com referências é uma prática comum para evitar cópias desnecessárias de objetos grandes enquanto garante que eles não sejam modificados.

### Parâmetros Padrão

C++ permite definir valores padrão para parâmetros, que são usados quando o argumento correspondente não é fornecido:

```cpp
void configurar_aimbot(float suavidade = 1.0f, bool apenas_cabeca = false, int fov = 90) {
    std::cout << "Aimbot configurado:" << std::endl;
    std::cout << "  Suavidade: " << suavidade << std::endl;
    std::cout << "  Apenas cabeça: " << (apenas_cabeca ? "Sim" : "Não") << std::endl;
    std::cout << "  FOV: " << fov << std::endl;
}

int main() {
    configurar_aimbot();  // Usa todos os valores padrão
    configurar_aimbot(2.5f);  // Suavidade = 2.5, resto padrão
    configurar_aimbot(1.5f, true);  // Suavidade = 1.5, Apenas cabeça = true, FOV padrão
    configurar_aimbot(3.0f, false, 120);  // Especifica todos os valores
    return 0;
}
```

Regras importantes para parâmetros padrão:
- Eles devem ser os últimos na lista de parâmetros
- Uma vez que um parâmetro tem valor padrão, todos os parâmetros à sua direita também devem ter valores padrão

## Retorno de Valores

As funções podem retornar valores usando a instrução `return`. O tipo de retorno é especificado na declaração da função:

```cpp
int somar(int a, int b) {
    return a + b;  // Retorna um int
}

bool esta_visivel(int jogador_id) {
    // Código para verificar visibilidade
    return true;  // ou false, dependendo da verificação
}

void mostrar_mensagem(const std::string& msg) {
    std::cout << msg << std::endl;
    // Sem return, pois o tipo de retorno é void
}
```

### Retornando Múltiplos Valores

C++ não permite retornar múltiplos valores diretamente, mas existem várias abordagens para contornar essa limitação:

1. **Usando uma estrutura ou classe**:
```cpp
struct Resultado {
    float distancia;
    bool visivel;
    int id_jogador;
};

Resultado encontrar_alvo_mais_proximo() {
    Resultado res;
    res.distancia = 150.5f;
    res.visivel = true;
    res.id_jogador = 42;
    return res;
}

int main() {
    Resultado alvo = encontrar_alvo_mais_proximo();
    std::cout << "Alvo encontrado: ID " << alvo.id_jogador 
              << ", Distância: " << alvo.distancia 
              << ", Visível: " << (alvo.visivel ? "Sim" : "Não") << std::endl;
    return 0;
}
```

2. **Usando parâmetros de saída** (por referência ou ponteiro):
```cpp
void encontrar_alvo(int& id_out, float& distancia_out, bool& visivel_out) {
    id_out = 42;
    distancia_out = 150.5f;
    visivel_out = true;
}

int main() {
    int id;
    float distancia;
    bool visivel;
    
    encontrar_alvo(id, distancia, visivel);
    
    std::cout << "Alvo encontrado: ID " << id 
              << ", Distância: " << distancia 
              << ", Visível: " << (visivel ? "Sim" : "Não") << std::endl;
    return 0;
}
```

3. **Usando std::tuple** (C++11 e posterior):
```cpp
#include <tuple>

std::tuple<int, float, bool> encontrar_alvo() {
    return std::make_tuple(42, 150.5f, true);
}

int main() {
    auto resultado = encontrar_alvo();
    
    int id = std::get<0>(resultado);
    float distancia = std::get<1>(resultado);
    bool visivel = std::get<2>(resultado);
    
    // Ou usando structured binding (C++17):
    // auto [id, distancia, visivel] = encontrar_alvo();
    
    std::cout << "Alvo encontrado: ID " << id 
              << ", Distância: " << distancia 
              << ", Visível: " << (visivel ? "Sim" : "Não") << std::endl;
    return 0;
}
```

4. **Usando std::pair** (para apenas dois valores):
```cpp
#include <utility>

std::pair<float, bool> verificar_alvo(int id) {
    return std::make_pair(150.5f, true);
}

int main() {
    auto resultado = verificar_alvo(42);
    float distancia = resultado.first;
    bool visivel = resultado.second;
    
    std::cout << "Alvo: Distância " << distancia 
              << ", Visível: " << (visivel ? "Sim" : "Não") << std::endl;
    return 0;
}
```

## Sobrecarga de Funções

C++ permite definir múltiplas funções com o mesmo nome, desde que tenham diferentes listas de parâmetros (número ou tipos diferentes). Isso é chamado de sobrecarga de funções:

```cpp
// Sobrecarga de funções
float calcular_dano(float dano_base, float multiplicador) {
    return dano_base * multiplicador;
}

float calcular_dano(float dano_base, float multiplicador, bool critico) {
    float dano = dano_base * multiplicador;
    if (critico) {
        dano *= 2.0f;
    }
    return dano;
}

int calcular_dano(int dano_base, int defesa) {
    return std::max(1, dano_base - defesa);  // Garante dano mínimo de 1
}
```

A sobrecarga de funções é útil para:
- Fornecer diferentes versões de uma função com diferentes níveis de complexidade
- Processar diferentes tipos de dados com a mesma operação lógica
- Criar APIs mais intuitivas e fáceis de usar

O compilador seleciona a função correta com base nos argumentos fornecidos na chamada.

## Funções Inline

A palavra-chave `inline` sugere ao compilador que substitua a chamada da função pelo seu corpo, evitando a sobrecarga de uma chamada de função real:

```cpp
inline float calcular_distancia_quadrada(float x1, float y1, float x2, float y2) {
    float dx = x2 - x1;
    float dy = y2 - y1;
    return dx*dx + dy*dy;
}
```

Funções inline são úteis para:
- Funções pequenas e frequentemente chamadas
- Situações onde o desempenho é crítico (como em cheats que precisam ser executados a cada frame)

No entanto, é importante notar que `inline` é apenas uma sugestão ao compilador, que pode ignorá-la se achar que a função é muito grande ou complexa para ser expandida inline.

## Exemplo Prático para Desenvolvimento de Cheats

Vamos ver um exemplo mais completo que demonstra o uso de funções em um contexto de desenvolvimento de cheats para jogos:

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <string>

// Estrutura para representar um jogador
struct Jogador {
    int id;
    std::string nome;
    float posicao_x;
    float posicao_y;
    float posicao_z;
    int vida;
    int equipe;  // 1 = aliado, 2 = inimigo
    bool visivel;
};

// Estrutura para representar um resultado de busca
struct ResultadoBusca {
    bool sucesso;
    int indice_jogador;
    float distancia;
    bool visivel;
};

// Calcula a distância entre dois pontos 2D
inline float calcular_distancia_2d(float x1, float y1, float x2, float y2) {
    float dx = x2 - x1;
    float dy = y2 - y1;
    return std::sqrt(dx*dx + dy*dy);
}

// Calcula a distância entre dois pontos 3D
inline float calcular_distancia_3d(float x1, float y1, float z1, float x2, float y2, float z2) {
    float dx = x2 - x1;
    float dy = y2 - y1;
    float dz = z2 - z1;
    return std::sqrt(dx*dx + dy*dy + dz*dz);
}

// Sobrecarga: Calcula a distância entre dois jogadores
float calcular_distancia(const Jogador& jogador1, const Jogador& jogador2) {
    return calcular_distancia_3d(
        jogador1.posicao_x, jogador1.posicao_y, jogador1.posicao_z,
        jogador2.posicao_x, jogador2.posicao_y, jogador2.posicao_z
    );
}

// Encontra o jogador inimigo mais próximo
ResultadoBusca encontrar_inimigo_mais_proximo(
    const std::vector<Jogador>& jogadores, 
    int minha_equipe, 
    float posicao_x, 
    float posicao_y, 
    float posicao_z,
    bool apenas_visiveis = true
) {
    ResultadoBusca resultado;
    resultado.sucesso = false;
    resultado.distancia = 999999.0f;  // Valor inicial alto
    
    for (size_t i = 0; i < jogadores.size(); i++) {
        const Jogador& jogador = jogadores[i];
        
        // Pula jogadores da mesma equipe
        if (jogador.equipe == minha_equipe) {
            continue;
        }
        
        // Pula jogadores não visíveis se apenas_visiveis for true
        if (apenas_visiveis && !jogador.visivel) {
            continue;
        }
        
        // Calcula a distância
        float distancia = calcular_distancia_3d(
            posicao_x, posicao_y, posicao_z,
            jogador.posicao_x, jogador.posicao_y, jogador.posicao_z
        );
        
        // Verifica se este jogador está mais próximo que o atual mais próximo
        if (distancia < resultado.distancia) {
            resultado.sucesso = true;
            resultado.indice_jogador = i;
            resultado.distancia = distancia;
            resultado.visivel = jogador.visivel;
        }
    }
    
    return resultado;
}

// Calcula o ângulo necessário para mirar em um alvo
void calcular_angulos_mira(
    float origem_x, float origem_y, float origem_z,
    float alvo_x, float alvo_y, float alvo_z,
    float& angulo_horizontal_out, float& angulo_vertical_out
) {
    float dx = alvo_x - origem_x;
    float dy = alvo_y - origem_y;
    float dz = alvo_z - origem_z;
    
    // Calcula o ângulo horizontal (yaw)
    angulo_horizontal_out = std::atan2(dy, dx) * 180.0f / M_PI;
    
    // Calcula o ângulo vertical (pitch)
    float distancia_horizontal = std::sqrt(dx*dx + dy*dy);
    angulo_vertical_out = std::atan2(-dz, distancia_horizontal) * 180.0f / M_PI;
}

// Função para ativar/desativar o aimbot
void configurar_aimbot(bool ativar, float suavidade = 1.0f, bool apenas_cabeca = false) {
    std::cout << "Aimbot " << (ativar ? "ativado" : "desativado") << std::endl;
    
    if (ativar) {
        std::cout << "  Suavidade: " << suavidade << std::endl;
        std::cout << "  Mirar apenas na cabeça: " << (apenas_cabeca ? "Sim" : "Não") << std::endl;
    }
}

// Função para desenhar ESP (Extra Sensory Perception)
void desenhar_esp(const Jogador& jogador, bool mostrar_vida = true, bool mostrar_distancia = true) {
    std::cout << "Desenhando ESP para " << jogador.nome << std::endl;
    
    if (mostrar_vida) {
        std::cout << "  Vida: " << jogador.vida << "/100" << std::endl;
    }
    
    if (mostrar_distancia) {
        std::cout << "  Distância: " << calcular_distancia_3d(
            0, 0, 0,  // Posição simulada do jogador local
            jogador.posicao_x, jogador.posicao_y, jogador.posicao_z
        ) << " unidades" << std::endl;
    }
}

// Função principal para demonstração
int main() {
    // Cria uma lista de jogadores simulados
    std::vector<Jogador> jogadores = {
        {1, "Você", 0, 0, 0, 100, 1, true},
        {2, "Aliado1", 10, 15, 0, 80, 1, true},
        {3, "Inimigo1", 50, 30, 0, 90, 2, true},
        {4, "Inimigo2", 100, 120, 0, 75, 2, false},
        {5, "Inimigo3", 20, 25, 0, 60, 2, true}
    };
    
    int minha_equipe = 1;
    
    // Encontra o inimigo mais próximo
    ResultadoBusca resultado = encontrar_inimigo_mais_proximo(
        jogadores, minha_equipe, 0, 0, 0, true
    );
    
    if (resultado.sucesso) {
        const Jogador& inimigo = jogadores[resultado.indice_jogador];
        
        std::cout << "Inimigo mais próximo: " << inimigo.nome << std::endl;
        std::cout << "Distância: " << resultado.distancia << std::endl;
        
        // Calcula ângulos de mira
        float angulo_horizontal, angulo_vertical;
        calcular_angulos_mira(
            0, 0, 0,  // Posição do jogador local
            inimigo.posicao_x, inimigo.posicao_y, inimigo.posicao_z,
            angulo_horizontal, angulo_vertical
        );
        
        std::
(Content truncated due to size limit. Use line ranges to read in chunks)