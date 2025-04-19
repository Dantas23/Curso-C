# Exemplo Prático 3: Hooking de Funções em Jogos

## Introdução

O hooking de funções é uma técnica avançada e poderosa no desenvolvimento de cheats para jogos, permitindo interceptar chamadas de funções e modificar seu comportamento. Diferente da simples leitura e escrita de memória, o hooking permite alterar a lógica do jogo, possibilitando modificações mais profundas e sofisticadas.

Neste exemplo prático, vamos explorar diferentes métodos de hooking e implementar um sistema que pode ser usado para modificar o comportamento de funções em jogos. Aprenderemos:

- O que é hooking e como ele funciona
- Diferentes métodos de hooking (IAT hooking, inline hooking, VTable hooking)
- Como implementar hooks em funções específicas
- Como criar trampolins (trampolines) para preservar a funcionalidade original
- Aplicações práticas de hooking em cheats para jogos

Este exemplo é específico para sistemas Windows e requer um bom entendimento de C++ e assembly de baixo nível.

## Requisitos

- Visual Studio 2022
- Sistema operacional Windows
- Conhecimentos avançados de C++ (ponteiros, funções, classes)
- Conhecimentos básicos de assembly x86/x64
- Compreensão de como as funções são chamadas em nível de máquina

## Parte 1: Biblioteca de Hooking

Primeiro, vamos criar uma biblioteca de hooking que implementará diferentes métodos de hooking. Esta biblioteca será a base para nossos exemplos práticos.

```cpp
// Hooking.h
#pragma once

#include <Windows.h>
#include <vector>
#include <string>
#include <memory>
#include <unordered_map>

// Classe para gerenciar hooks
class HookManager {
private:
    // Estrutura para armazenar informações sobre um hook
    struct HookInfo {
        void* originalFunction;
        void* hookFunction;
        void* trampolineFunction;
        size_t hookSize;
        std::vector<BYTE> originalBytes;
        bool enabled;
    };
    
    // Mapa de hooks instalados
    std::unordered_map<void*, HookInfo> hooks;
    
    // Singleton
    static HookManager* instance;
    HookManager() {}
    
public:
    // Obtém a instância do singleton
    static HookManager* GetInstance() {
        if (!instance) {
            instance = new HookManager();
        }
        return instance;
    }
    
    // Libera a instância
    static void ReleaseInstance() {
        if (instance) {
            delete instance;
            instance = nullptr;
        }
    }
    
    // Cria um hook inline
    bool CreateInlineHook(void* targetFunction, void* hookFunction, void** originalFunction);
    
    // Remove um hook inline
    bool RemoveInlineHook(void* targetFunction);
    
    // Habilita/desabilita um hook
    bool EnableHook(void* targetFunction, bool enable);
    
    // Cria um hook de IAT (Import Address Table)
    bool CreateIATHook(const char* moduleName, const char* functionName, void* hookFunction, void** originalFunction);
    
    // Cria um hook de VTable
    bool CreateVTableHook(void* classInstance, int functionIndex, void* hookFunction, void** originalFunction);
};

// Classe para facilitar a criação de hooks
template<typename T>
class Hook {
private:
    void* targetFunction;
    T hookFunction;
    T originalFunction;
    bool installed;
    
public:
    Hook() : targetFunction(nullptr), hookFunction(nullptr), originalFunction(nullptr), installed(false) {}
    
    // Instala um hook inline
    bool Install(void* target, T hook) {
        targetFunction = target;
        hookFunction = hook;
        
        if (HookManager::GetInstance()->CreateInlineHook(
            targetFunction, 
            reinterpret_cast<void*>(hookFunction), 
            reinterpret_cast<void**>(&originalFunction))) {
            installed = true;
            return true;
        }
        
        return false;
    }
    
    // Instala um hook de IAT
    bool InstallIAT(const char* moduleName, const char* functionName, T hook) {
        hookFunction = hook;
        
        if (HookManager::GetInstance()->CreateIATHook(
            moduleName, 
            functionName, 
            reinterpret_cast<void*>(hookFunction), 
            reinterpret_cast<void**>(&originalFunction))) {
            installed = true;
            return true;
        }
        
        return false;
    }
    
    // Instala um hook de VTable
    bool InstallVTable(void* classInstance, int functionIndex, T hook) {
        hookFunction = hook;
        
        if (HookManager::GetInstance()->CreateVTableHook(
            classInstance, 
            functionIndex, 
            reinterpret_cast<void*>(hookFunction), 
            reinterpret_cast<void**>(&originalFunction))) {
            installed = true;
            return true;
        }
        
        return false;
    }
    
    // Remove o hook
    bool Remove() {
        if (!installed) {
            return false;
        }
        
        if (HookManager::GetInstance()->RemoveInlineHook(targetFunction)) {
            installed = false;
            return true;
        }
        
        return false;
    }
    
    // Habilita/desabilita o hook
    bool Enable(bool enable) {
        if (!installed) {
            return false;
        }
        
        return HookManager::GetInstance()->EnableHook(targetFunction, enable);
    }
    
    // Obtém a função original
    T GetOriginal() const {
        return originalFunction;
    }
    
    // Verifica se o hook está instalado
    bool IsInstalled() const {
        return installed;
    }
};

// Inicializa o singleton
HookManager* HookManager::instance = nullptr;
```

Agora, vamos implementar os métodos da classe `HookManager`:

```cpp
// Hooking.cpp
#include "Hooking.h"
#include <Psapi.h>

// Tamanho mínimo para um hook inline (5 bytes para jmp)
constexpr size_t MIN_HOOK_SIZE = 5;

// Função para obter o tamanho de uma instrução
size_t GetInstructionSize(void* address) {
    // Esta é uma implementação simplificada
    // Em um cenário real, você precisaria de um disassembler como Zydis ou Capstone
    // para determinar corretamente o tamanho das instruções
    
    // Para este exemplo, vamos assumir que todas as instruções têm pelo menos 5 bytes
    return MIN_HOOK_SIZE;
}

// Cria um hook inline
bool HookManager::CreateInlineHook(void* targetFunction, void* hookFunction, void** originalFunction) {
    // Verifica se o hook já existe
    if (hooks.find(targetFunction) != hooks.end()) {
        return false;
    }
    
    // Determina o tamanho necessário para o hook
    size_t hookSize = 0;
    size_t totalSize = 0;
    
    // Precisamos de pelo menos 5 bytes para um jmp
    while (totalSize < MIN_HOOK_SIZE) {
        size_t instructionSize = GetInstructionSize((void*)((uintptr_t)targetFunction + totalSize));
        totalSize += instructionSize;
    }
    
    hookSize = totalSize;
    
    // Cria um trampoline para a função original
    void* trampolineFunction = VirtualAlloc(NULL, hookSize + MIN_HOOK_SIZE, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (!trampolineFunction) {
        return false;
    }
    
    // Salva os bytes originais
    std::vector<BYTE> originalBytes(hookSize);
    memcpy(originalBytes.data(), targetFunction, hookSize);
    
    // Copia os bytes originais para o trampoline
    memcpy(trampolineFunction, targetFunction, hookSize);
    
    // Adiciona um jmp de volta para a função original após o hook
    BYTE* trampolineJmp = (BYTE*)trampolineFunction + hookSize;
    *trampolineJmp = 0xE9; // jmp
    *(DWORD*)(trampolineJmp + 1) = (DWORD)((uintptr_t)targetFunction + hookSize - ((uintptr_t)trampolineJmp + 5));
    
    // Modifica a proteção da memória da função alvo
    DWORD oldProtect;
    if (!VirtualProtect(targetFunction, hookSize, PAGE_EXECUTE_READWRITE, &oldProtect)) {
        VirtualFree(trampolineFunction, 0, MEM_RELEASE);
        return false;
    }
    
    // Escreve o hook (jmp para a função de hook)
    memset(targetFunction, 0x90, hookSize); // Preenche com NOPs
    *(BYTE*)targetFunction = 0xE9; // jmp
    *(DWORD*)((uintptr_t)targetFunction + 1) = (DWORD)((uintptr_t)hookFunction - ((uintptr_t)targetFunction + 5));
    
    // Restaura a proteção da memória
    VirtualProtect(targetFunction, hookSize, oldProtect, &oldProtect);
    
    // Armazena as informações do hook
    HookInfo info;
    info.originalFunction = targetFunction;
    info.hookFunction = hookFunction;
    info.trampolineFunction = trampolineFunction;
    info.hookSize = hookSize;
    info.originalBytes = originalBytes;
    info.enabled = true;
    
    hooks[targetFunction] = info;
    
    // Define o ponteiro para a função original
    *originalFunction = trampolineFunction;
    
    return true;
}

// Remove um hook inline
bool HookManager::RemoveInlineHook(void* targetFunction) {
    // Verifica se o hook existe
    auto it = hooks.find(targetFunction);
    if (it == hooks.end()) {
        return false;
    }
    
    const HookInfo& info = it->second;
    
    // Modifica a proteção da memória
    DWORD oldProtect;
    if (!VirtualProtect(targetFunction, info.hookSize, PAGE_EXECUTE_READWRITE, &oldProtect)) {
        return false;
    }
    
    // Restaura os bytes originais
    memcpy(targetFunction, info.originalBytes.data(), info.hookSize);
    
    // Restaura a proteção da memória
    VirtualProtect(targetFunction, info.hookSize, oldProtect, &oldProtect);
    
    // Libera o trampoline
    VirtualFree(info.trampolineFunction, 0, MEM_RELEASE);
    
    // Remove o hook do mapa
    hooks.erase(it);
    
    return true;
}

// Habilita/desabilita um hook
bool HookManager::EnableHook(void* targetFunction, bool enable) {
    // Verifica se o hook existe
    auto it = hooks.find(targetFunction);
    if (it == hooks.end()) {
        return false;
    }
    
    HookInfo& info = it->second;
    
    // Se o estado já é o desejado, não faz nada
    if (info.enabled == enable) {
        return true;
    }
    
    // Modifica a proteção da memória
    DWORD oldProtect;
    if (!VirtualProtect(targetFunction, info.hookSize, PAGE_EXECUTE_READWRITE, &oldProtect)) {
        return false;
    }
    
    if (enable) {
        // Habilita o hook
        memset(targetFunction, 0x90, info.hookSize); // Preenche com NOPs
        *(BYTE*)targetFunction = 0xE9; // jmp
        *(DWORD*)((uintptr_t)targetFunction + 1) = (DWORD)((uintptr_t)info.hookFunction - ((uintptr_t)targetFunction + 5));
    } else {
        // Desabilita o hook
        memcpy(targetFunction, info.originalBytes.data(), info.hookSize);
    }
    
    // Restaura a proteção da memória
    VirtualProtect(targetFunction, info.hookSize, oldProtect, &oldProtect);
    
    // Atualiza o estado
    info.enabled = enable;
    
    return true;
}

// Função auxiliar para encontrar o endereço de uma função importada
void* FindIATFunction(HMODULE module, const char* importModule, const char* functionName) {
    if (!module || !importModule || !functionName) {
        return nullptr;
    }
    
    // Obtém o cabeçalho do DOS
    PIMAGE_DOS_HEADER dosHeader = (PIMAGE_DOS_HEADER)module;
    if (dosHeader->e_magic != IMAGE_DOS_SIGNATURE) {
        return nullptr;
    }
    
    // Obtém o cabeçalho do NT
    PIMAGE_NT_HEADERS ntHeaders = (PIMAGE_NT_HEADERS)((BYTE*)module + dosHeader->e_lfanew);
    if (ntHeaders->Signature != IMAGE_NT_SIGNATURE) {
        return nullptr;
    }
    
    // Obtém o diretório de importação
    PIMAGE_IMPORT_DESCRIPTOR importDesc = (PIMAGE_IMPORT_DESCRIPTOR)((BYTE*)module + 
        ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);
    
    // Percorre os módulos importados
    for (; importDesc->Name; importDesc++) {
        const char* moduleName = (const char*)((BYTE*)module + importDesc->Name);
        
        // Verifica se é o módulo que estamos procurando
        if (_stricmp(moduleName, importModule) == 0) {
            // Obtém a tabela de endereços de importação (IAT)
            PIMAGE_THUNK_DATA thunk = (PIMAGE_THUNK_DATA)((BYTE*)module + importDesc->FirstThunk);
            PIMAGE_THUNK_DATA originalThunk = (PIMAGE_THUNK_DATA)((BYTE*)module + importDesc->OriginalFirstThunk);
            
            // Percorre as funções importadas
            for (; originalThunk->u1.AddressOfData; originalThunk++, thunk++) {
                // Verifica se a função é importada por nome
                if (originalThunk->u1.Ordinal & IMAGE_ORDINAL_FLAG) {
                    continue; // Importada por ordinal, não por nome
                }
                
                // Obtém o nome da função
                PIMAGE_IMPORT_BY_NAME importByName = (PIMAGE_IMPORT_BY_NAME)((BYTE*)module + 
                    originalThunk->u1.AddressOfData);
                
                // Verifica se é a função que estamos procurando
                if (strcmp((const char*)importByName->Name, functionName) == 0) {
                    return &thunk->u1.Function;
                }
            }
            
            break;
        }
    }
    
    return nullptr;
}

// Cria um hook de IAT
bool HookManager::CreateIATHook(const char* moduleName, const char* functionName, void* hookFunction, void** originalFunction) {
    // Obtém o módulo base do processo
    HMODULE baseModule = GetModuleHandleA(NULL);
    
    // Encontra o endereço da função na IAT
    void** iatEntry = (void**)FindIATFunction(baseModule, moduleName, functionName);
    if (!iatEntry) {
        return false;
    }
    
    // Salva o endereço original
    *originalFunction = *iatEntry;
    
    // Modifica a proteção da memória
    DWORD oldProtect;
    if (!VirtualProtect(iatEntry, sizeof(void*), PAGE_READWRITE, &oldProtect)) {
        return false;
    }
    
    // Substitui o endereço na IAT
    *iatEntry = hookFunction;
    
    // Restaura a proteção da memória
    VirtualProtect(iatEntry, sizeof(void*), oldProtect, &oldProtect);
    
    return true;
}

// Cria um hook de VTable
bool HookManager::CreateVTableHook(void* classInstance, int functionIndex, void* hookFunction, void** originalFunction) {
    if (!classInstance) {
        return false;
    }
    
    // Obtém o ponteiro para a VTable
    void** vtable = *(void***)classInstance;
    
    // Salva o endereço original
    *originalFunction = vtable[functionIndex];
    
    // Modifica a proteção da memória
    DWORD oldProtect;
    if (!VirtualProtect(&vtable[functionIndex], sizeof(void*), PAGE_READWRITE, &oldProtect)) {
        return false;
    }
    
    // Substitui o endereço na VTable
    vtable[functionIndex] = hookFunction;
    
    // Restaura a proteção da memória
    VirtualProtect(&vtable[functionIndex], sizeof(void*), oldProtect, &oldProtect);
    
    return true;
}
```

## Parte 2: Exemplo de Aplicação - Hooking de DirectX

Agora, vamos criar um exemplo prático que demonstra como usar nossa biblioteca de hooking para interceptar funções do DirectX, permitindo implementar um ESP (Extra Sensory Perception) básico.

```cpp
// DirectXHook.cpp
#include <Windows.h>
#include <d3d11.h>
#include <dxgi.h>
#include <vector>
#include <string>
#include <iostream>
#include <fstream>
#include "Hooking.h"

// Protótipos das funções originais
typedef HRESULT (WINAPI* D3D11Present)(IDXGISwapChain* pSwapChain, UINT SyncInterval, UINT Flags);
typedef HRESULT (WINAPI* D3D11DrawIndexed)(ID3D11DeviceContext* pContext, UINT IndexCount, UINT StartIndexLocation, INT BaseVertexLocation);

// Hooks
Hook<D3D11Present> g_presentHook;
Hook<D3D11DrawIndexed> g_drawIndexedHook;

// Interfaces do DirectX
ID3D11Device* g_pd3dDevice = nullptr;
ID3D11DeviceContext* g_pd3dContext = nullptr;
IDXGISwapChain* g_pSwapChain = nullptr;
ID3D11RenderTargetView* g_mainRenderTargetView = nullptr;

// Variáveis de configuração
bool g_bESPEnabled = true;
bool g_bWallhackEnabled = false;

// Arquivo de log
std::ofstream g_logFile;

// Função para inicializar o log
void InitLog() {
    g_logFile.open("directx_hook.log");
    if (g_logFile.is_open()) {
        g_logFile << "DirectX Hook inicializado
(Content truncated due to size limit. Use line ranges to read in chunks)