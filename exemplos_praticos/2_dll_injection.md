# Exemplo Prático 2: Injeção de DLL em Processos

## Introdução

A injeção de DLL é uma técnica fundamental no desenvolvimento de cheats para jogos, permitindo executar código personalizado dentro do espaço de memória de um processo em execução. Isso possibilita modificar o comportamento do jogo de maneiras que não seriam possíveis apenas com a leitura e escrita de memória.

Neste exemplo prático, vamos criar um injetor de DLL básico e uma DLL de exemplo que, quando injetada em um processo, poderá modificar seu comportamento. Aprenderemos:

- Como criar uma DLL com funções que serão executadas quando a DLL for carregada
- Como injetar essa DLL em um processo em execução
- Como implementar funcionalidades básicas dentro da DLL injetada
- Como estabelecer comunicação entre o processo injetor e a DLL injetada

Este exemplo é específico para sistemas Windows e utiliza a API do Windows para realizar a injeção.

## Requisitos

- Visual Studio 2022
- Sistema operacional Windows
- Conhecimentos intermediários de C++ (classes, ponteiros, funções)
- Compreensão básica de como os processos e DLLs funcionam no Windows

## Parte 1: Criando o Injetor de DLL

Primeiro, vamos criar o programa que será responsável por injetar nossa DLL em um processo alvo. Crie um novo projeto de aplicativo de console no Visual Studio e adicione o seguinte código:

```cpp
// DLL_Injector.cpp
#include <iostream>
#include <Windows.h>
#include <TlHelp32.h>
#include <string>
#include <vector>

// Função para obter o ID de um processo pelo nome
DWORD GetProcessIdByName(const std::string& processName) {
    DWORD processId = 0;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    
    if (snapshot != INVALID_HANDLE_VALUE) {
        PROCESSENTRY32 processEntry;
        processEntry.dwSize = sizeof(PROCESSENTRY32);
        
        if (Process32First(snapshot, &processEntry)) {
            do {
                // Converte o nome do processo para string para comparação
                std::string currentProcessName = processEntry.szExeFile;
                
                // Compara os nomes de processo (case-insensitive)
                if (_stricmp(currentProcessName.c_str(), processName.c_str()) == 0) {
                    processId = processEntry.th32ProcessID;
                    break;
                }
            } while (Process32Next(snapshot, &processEntry));
        }
        
        CloseHandle(snapshot);
    }
    
    return processId;
}

// Função para listar todos os processos em execução
std::vector<std::pair<DWORD, std::string>> ListProcesses() {
    std::vector<std::pair<DWORD, std::string>> processes;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    
    if (snapshot != INVALID_HANDLE_VALUE) {
        PROCESSENTRY32 processEntry;
        processEntry.dwSize = sizeof(PROCESSENTRY32);
        
        if (Process32First(snapshot, &processEntry)) {
            do {
                processes.push_back(std::make_pair(
                    processEntry.th32ProcessID,
                    std::string(processEntry.szExeFile)
                ));
            } while (Process32Next(snapshot, &processEntry));
        }
        
        CloseHandle(snapshot);
    }
    
    return processes;
}

// Função para injetar uma DLL em um processo
bool InjectDLL(DWORD processId, const std::string& dllPath) {
    // Abre o processo alvo
    HANDLE processHandle = OpenProcess(
        PROCESS_CREATE_THREAD | PROCESS_QUERY_INFORMATION | PROCESS_VM_OPERATION | PROCESS_VM_WRITE | PROCESS_VM_READ,
        FALSE,
        processId
    );
    
    if (processHandle == NULL) {
        std::cerr << "Erro ao abrir o processo. Código de erro: " << GetLastError() << std::endl;
        return false;
    }
    
    // Aloca memória no processo alvo para o caminho da DLL
    LPVOID dllPathAddress = VirtualAllocEx(
        processHandle,
        NULL,
        dllPath.length() + 1,
        MEM_COMMIT | MEM_RESERVE,
        PAGE_READWRITE
    );
    
    if (dllPathAddress == NULL) {
        std::cerr << "Erro ao alocar memória no processo. Código de erro: " << GetLastError() << std::endl;
        CloseHandle(processHandle);
        return false;
    }
    
    // Escreve o caminho da DLL na memória alocada
    if (!WriteProcessMemory(
        processHandle,
        dllPathAddress,
        dllPath.c_str(),
        dllPath.length() + 1,
        NULL
    )) {
        std::cerr << "Erro ao escrever na memória do processo. Código de erro: " << GetLastError() << std::endl;
        VirtualFreeEx(processHandle, dllPathAddress, 0, MEM_RELEASE);
        CloseHandle(processHandle);
        return false;
    }
    
    // Obtém o endereço da função LoadLibraryA
    HMODULE kernel32Module = GetModuleHandleA("kernel32.dll");
    if (kernel32Module == NULL) {
        std::cerr << "Erro ao obter o handle do kernel32.dll. Código de erro: " << GetLastError() << std::endl;
        VirtualFreeEx(processHandle, dllPathAddress, 0, MEM_RELEASE);
        CloseHandle(processHandle);
        return false;
    }
    
    FARPROC loadLibraryAddress = GetProcAddress(kernel32Module, "LoadLibraryA");
    if (loadLibraryAddress == NULL) {
        std::cerr << "Erro ao obter o endereço de LoadLibraryA. Código de erro: " << GetLastError() << std::endl;
        VirtualFreeEx(processHandle, dllPathAddress, 0, MEM_RELEASE);
        CloseHandle(processHandle);
        return false;
    }
    
    // Cria uma thread remota que chama LoadLibraryA com o caminho da DLL
    HANDLE remoteThread = CreateRemoteThread(
        processHandle,
        NULL,
        0,
        (LPTHREAD_START_ROUTINE)loadLibraryAddress,
        dllPathAddress,
        0,
        NULL
    );
    
    if (remoteThread == NULL) {
        std::cerr << "Erro ao criar thread remota. Código de erro: " << GetLastError() << std::endl;
        VirtualFreeEx(processHandle, dllPathAddress, 0, MEM_RELEASE);
        CloseHandle(processHandle);
        return false;
    }
    
    // Espera a thread terminar
    WaitForSingleObject(remoteThread, INFINITE);
    
    // Obtém o código de retorno da thread (handle da DLL ou 0 se falhou)
    DWORD exitCode;
    GetExitCodeThread(remoteThread, &exitCode);
    
    // Limpa
    CloseHandle(remoteThread);
    VirtualFreeEx(processHandle, dllPathAddress, 0, MEM_RELEASE);
    CloseHandle(processHandle);
    
    return (exitCode != 0);
}

int main() {
    std::cout << "=== Injetor de DLL ===" << std::endl;
    std::cout << "Este programa permite injetar uma DLL em um processo em execução." << std::endl;
    std::cout << "AVISO: Use este programa apenas para fins educacionais e em processos que você tem permissão para modificar." << std::endl;
    std::cout << std::endl;
    
    while (true) {
        std::cout << "\n=== Menu Principal ===" << std::endl;
        std::cout << "1. Listar processos" << std::endl;
        std::cout << "2. Injetar DLL (por ID do processo)" << std::endl;
        std::cout << "3. Injetar DLL (por nome do processo)" << std::endl;
        std::cout << "4. Sair" << std::endl;
        std::cout << "\nEscolha uma opção: ";
        
        int option;
        std::cin >> option;
        
        if (std::cin.fail()) {
            std::cin.clear();
            std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
            std::cout << "Entrada inválida. Tente novamente." << std::endl;
            continue;
        }
        
        switch (option) {
            case 1: {
                // Listar processos
                std::cout << "\nProcessos em execução:" << std::endl;
                std::cout << "ID\t| Nome" << std::endl;
                std::cout << "------------------------" << std::endl;
                
                auto processes = ListProcesses();
                for (const auto& process : processes) {
                    std::cout << process.first << "\t| " << process.second << std::endl;
                }
                break;
            }
            
            case 2: {
                // Injetar DLL por ID do processo
                std::cout << "Digite o ID do processo: ";
                DWORD processId;
                std::cin >> processId;
                
                if (std::cin.fail()) {
                    std::cin.clear();
                    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                    std::cout << "ID inválido." << std::endl;
                    break;
                }
                
                std::cin.ignore(); // Limpa o buffer
                
                std::cout << "Digite o caminho completo da DLL: ";
                std::string dllPath;
                std::getline(std::cin, dllPath);
                
                std::cout << "Injetando DLL no processo " << processId << "..." << std::endl;
                
                if (InjectDLL(processId, dllPath)) {
                    std::cout << "DLL injetada com sucesso!" << std::endl;
                } else {
                    std::cout << "Falha ao injetar a DLL." << std::endl;
                }
                break;
            }
            
            case 3: {
                // Injetar DLL por nome do processo
                std::cin.ignore(); // Limpa o buffer
                
                std::cout << "Digite o nome do processo (ex: notepad.exe): ";
                std::string processName;
                std::getline(std::cin, processName);
                
                DWORD processId = GetProcessIdByName(processName);
                if (processId == 0) {
                    std::cout << "Processo não encontrado." << std::endl;
                    break;
                }
                
                std::cout << "Processo encontrado. ID: " << processId << std::endl;
                
                std::cout << "Digite o caminho completo da DLL: ";
                std::string dllPath;
                std::getline(std::cin, dllPath);
                
                std::cout << "Injetando DLL no processo " << processName << " (ID: " << processId << ")..." << std::endl;
                
                if (InjectDLL(processId, dllPath)) {
                    std::cout << "DLL injetada com sucesso!" << std::endl;
                } else {
                    std::cout << "Falha ao injetar a DLL." << std::endl;
                }
                break;
            }
            
            case 4: {
                // Sair
                std::cout << "Encerrando o programa..." << std::endl;
                return 0;
            }
            
            default: {
                std::cout << "Opção inválida." << std::endl;
                break;
            }
        }
    }
    
    return 0;
}
```

## Parte 2: Criando a DLL de Exemplo

Agora, vamos criar a DLL que será injetada no processo alvo. Crie um novo projeto de DLL no Visual Studio e adicione o seguinte código:

```cpp
// Cheat_DLL.cpp
#include <Windows.h>
#include <iostream>
#include <fstream>
#include <thread>
#include <vector>
#include <string>
#include <TlHelp32.h>

// Variáveis globais
HMODULE g_hModule = NULL;
bool g_bRunning = true;
std::ofstream g_logFile;

// Função para criar uma console para debug
void CreateDebugConsole() {
    AllocConsole();
    FILE* pFile = nullptr;
    freopen_s(&pFile, "CONOUT$", "w", stdout);
    std::cout << "DLL injetada com sucesso!" << std::endl;
    std::cout << "Console de debug inicializada." << std::endl;
}

// Função para iniciar o log em arquivo
void InitializeLog() {
    char modulePath[MAX_PATH];
    GetModuleFileNameA(g_hModule, modulePath, MAX_PATH);
    
    std::string logPath = modulePath;
    size_t lastSlash = logPath.find_last_of("\\/");
    if (lastSlash != std::string::npos) {
        logPath = logPath.substr(0, lastSlash + 1);
    }
    logPath += "cheat_log.txt";
    
    g_logFile.open(logPath);
    if (g_logFile.is_open()) {
        g_logFile << "DLL injetada com sucesso!" << std::endl;
        g_logFile << "Log inicializado em: " << logPath << std::endl;
    }
}

// Função para escrever no log
void Log(const std::string& message) {
    if (g_logFile.is_open()) {
        g_logFile << message << std::endl;
        g_logFile.flush();
    }
}

// Função para encontrar um padrão de bytes na memória
uintptr_t FindPattern(uintptr_t startAddress, uintptr_t endAddress, const std::vector<BYTE>& pattern, const std::string& mask) {
    for (uintptr_t i = startAddress; i < endAddress - pattern.size(); i++) {
        bool found = true;
        for (size_t j = 0; j < pattern.size(); j++) {
            if (mask[j] == 'x' && *(BYTE*)(i + j) != pattern[j]) {
                found = false;
                break;
            }
        }
        
        if (found) {
            return i;
        }
    }
    
    return 0;
}

// Função para encontrar um módulo pelo nome
MODULEINFO GetModuleInfo(const char* moduleName) {
    MODULEINFO modInfo = {0};
    HMODULE hModule = GetModuleHandleA(moduleName);
    
    if (hModule) {
        GetModuleInformation(GetCurrentProcess(), hModule, &modInfo, sizeof(MODULEINFO));
    }
    
    return modInfo;
}

// Função para modificar a vida do jogador (exemplo)
void ModifyPlayerHealth() {
    // Este é apenas um exemplo. Em um jogo real, você precisaria encontrar
    // o endereço correto da vida do jogador, que pode variar de jogo para jogo.
    
    // Exemplo: Procura por um padrão de bytes que possa indicar a estrutura do jogador
    MODULEINFO mainModule = GetModuleInfo(NULL); // Módulo principal do processo
    
    if (mainModule.lpBaseOfDll) {
        uintptr_t baseAddress = (uintptr_t)mainModule.lpBaseOfDll;
        uintptr_t moduleSize = mainModule.SizeOfImage;
        
        // Exemplo de padrão (isso é fictício e precisaria ser adaptado para um jogo real)
        std::vector<BYTE> pattern = {0x89, 0x86, 0x00, 0x00, 0x00, 0x00, 0x85, 0xC0};
        std::string mask = "xx????xx";
        
        uintptr_t patternAddress = FindPattern(baseAddress, baseAddress + moduleSize, pattern, mask);
        
        if (patternAddress) {
            Log("Padrão encontrado em: 0x" + std::to_string(patternAddress));
            
            // Exemplo: Lê o offset da vida a partir do padrão encontrado
            int healthOffset = *(int*)(patternAddress + 2);
            Log("Offset da vida: 0x" + std::to_string(healthOffset));
            
            // Exemplo: Modifica a vida do jogador
            // Isso é apenas um exemplo e não funcionará em um jogo real sem adaptação
            uintptr_t playerBase = *(uintptr_t*)(baseAddress + 0x10ABCDE); // Endereço fictício
            if (playerBase) {
                int* healthPtr = (int*)(playerBase + healthOffset);
                *healthPtr = 999; // Define a vida como 999
                Log("Vida modificada para 999");
            }
        } else {
            Log("Padrão não encontrado");
        }
    }
}

// Função para ativar/desativar munição infinita (exemplo)
void ToggleInfiniteAmmo(bool enable) {
    // Este é apenas um exemplo. Em um jogo real, você precisaria encontrar
    // o endereço correto da função que decrementa a munição.
    
    MODULEINFO mainModule = GetModuleInfo(NULL);
    
    if (mainModule.lpBaseOfDll) {
        uintptr_t baseAddress = (uintptr_t)mainModule.lpBaseOfDll;
        uintptr_t moduleSize = mainModule.SizeOfImage;
        
        // Exemplo de padrão para a função que decrementa munição (fictício)
        std::vector<BYTE> pattern = {0x89, 0x86, 0x00, 0x00, 0x00, 0x00, 0xFF, 0x8E};
        std::string mask = "xx????xx";
        
        uintptr_t patternAddress = FindPattern(baseAddress, baseAddress + moduleSize, pattern, mask);
        
        if (patternAddress) {
            Log("Padrão de munição encontrado em: 0x" + std::to_string(patternAddress));
            
            // Exemplo: Patch para pular a instrução que decrementa a munição
            if (enable) {
                // Substi
(Content truncated due to size limit. Use line ranges to read in chunks)