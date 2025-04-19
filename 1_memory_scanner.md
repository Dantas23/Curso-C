# Exemplo Prático 1: Scanner de Memória Básico

## Introdução

Neste exemplo prático, vamos criar um scanner de memória básico que pode ser usado para encontrar valores específicos na memória de um processo. Este é um dos fundamentos essenciais para o desenvolvimento de cheats para jogos, pois permite localizar onde as informações importantes (como vida, munição, dinheiro, etc.) estão armazenadas na memória do jogo.

O scanner de memória que vamos desenvolver terá as seguintes funcionalidades:
- Listar processos em execução
- Conectar-se a um processo específico
- Escanear a memória do processo em busca de valores específicos
- Realizar escaneamentos sucessivos para refinar os resultados
- Modificar valores encontrados na memória

Este exemplo utiliza a API do Windows para acessar a memória de outros processos, então é específico para sistemas Windows.

## Requisitos

- Visual Studio 2022
- Sistema operacional Windows
- Conhecimentos básicos de C++ (variáveis, funções, ponteiros)
- Compreensão básica de como os processos funcionam no Windows

## Código Completo

Vamos criar um arquivo chamado `memory_scanner.cpp` com o seguinte código:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <Windows.h>
#include <TlHelp32.h>
#include <iomanip>
#include <chrono>
#include <thread>

// Classe para gerenciar o acesso à memória de um processo
class MemoryManager {
private:
    HANDLE processHandle;
    DWORD processId;
    std::string processName;
    bool connected;

public:
    MemoryManager() : processHandle(NULL), processId(0), connected(false) {}

    ~MemoryManager() {
        disconnect();
    }

    // Lista todos os processos em execução
    std::vector<std::pair<DWORD, std::string>> listProcesses() {
        std::vector<std::pair<DWORD, std::string>> processes;
        
        HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (snapshot == INVALID_HANDLE_VALUE) {
            std::cerr << "Erro ao criar snapshot de processos" << std::endl;
            return processes;
        }

        PROCESSENTRY32 processEntry;
        processEntry.dwSize = sizeof(PROCESSENTRY32);

        if (Process32First(snapshot, &processEntry)) {
            do {
                processes.push_back(std::make_pair(processEntry.th32ProcessID, std::string(processEntry.szExeFile)));
            } while (Process32Next(snapshot, &processEntry));
        }

        CloseHandle(snapshot);
        return processes;
    }

    // Conecta-se a um processo pelo ID
    bool connectToProcess(DWORD pid) {
        disconnect(); // Desconecta de qualquer processo anterior

        processId = pid;
        processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processId);
        
        if (processHandle == NULL) {
            std::cerr << "Erro ao abrir o processo. Código de erro: " << GetLastError() << std::endl;
            std::cerr << "Tente executar como administrador." << std::endl;
            return false;
        }

        // Obtém o nome do processo
        HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (snapshot != INVALID_HANDLE_VALUE) {
            PROCESSENTRY32 processEntry;
            processEntry.dwSize = sizeof(PROCESSENTRY32);

            if (Process32First(snapshot, &processEntry)) {
                do {
                    if (processEntry.th32ProcessID == processId) {
                        processName = processEntry.szExeFile;
                        break;
                    }
                } while (Process32Next(snapshot, &processEntry));
            }
            CloseHandle(snapshot);
        }

        connected = true;
        return true;
    }

    // Conecta-se a um processo pelo nome
    bool connectToProcess(const std::string& name) {
        auto processes = listProcesses();
        for (const auto& process : processes) {
            if (process.second == name) {
                return connectToProcess(process.first);
            }
        }
        std::cerr << "Processo não encontrado: " << name << std::endl;
        return false;
    }

    // Desconecta-se do processo atual
    void disconnect() {
        if (processHandle != NULL) {
            CloseHandle(processHandle);
            processHandle = NULL;
        }
        processId = 0;
        processName = "";
        connected = false;
    }

    // Verifica se está conectado a um processo
    bool isConnected() const {
        return connected;
    }

    // Obtém o ID do processo conectado
    DWORD getProcessId() const {
        return processId;
    }

    // Obtém o nome do processo conectado
    std::string getProcessName() const {
        return processName;
    }

    // Lê um valor da memória do processo
    template <typename T>
    bool readMemory(uintptr_t address, T& value) {
        if (!connected) {
            std::cerr << "Não conectado a nenhum processo" << std::endl;
            return false;
        }

        SIZE_T bytesRead;
        if (!ReadProcessMemory(processHandle, reinterpret_cast<LPCVOID>(address), &value, sizeof(T), &bytesRead)) {
            return false;
        }

        return bytesRead == sizeof(T);
    }

    // Escreve um valor na memória do processo
    template <typename T>
    bool writeMemory(uintptr_t address, const T& value) {
        if (!connected) {
            std::cerr << "Não conectado a nenhum processo" << std::endl;
            return false;
        }

        SIZE_T bytesWritten;
        if (!WriteProcessMemory(processHandle, reinterpret_cast<LPVOID>(address), &value, sizeof(T), &bytesWritten)) {
            std::cerr << "Erro ao escrever na memória. Código de erro: " << GetLastError() << std::endl;
            return false;
        }

        return bytesWritten == sizeof(T);
    }

    // Obtém informações sobre as regiões de memória do processo
    std::vector<std::pair<uintptr_t, SIZE_T>> getMemoryRegions() {
        std::vector<std::pair<uintptr_t, SIZE_T>> regions;
        
        if (!connected) {
            std::cerr << "Não conectado a nenhum processo" << std::endl;
            return regions;
        }

        MEMORY_BASIC_INFORMATION mbi;
        uintptr_t address = 0;

        while (VirtualQueryEx(processHandle, reinterpret_cast<LPCVOID>(address), &mbi, sizeof(mbi))) {
            // Verifica se a região é acessível e não é uma região de sistema
            if ((mbi.State == MEM_COMMIT) && 
                (mbi.Protect == PAGE_READWRITE || mbi.Protect == PAGE_READONLY) && 
                (mbi.Type == MEM_PRIVATE || mbi.Type == MEM_MAPPED)) {
                regions.push_back(std::make_pair(reinterpret_cast<uintptr_t>(mbi.BaseAddress), mbi.RegionSize));
            }
            
            // Avança para a próxima região
            address = reinterpret_cast<uintptr_t>(mbi.BaseAddress) + mbi.RegionSize;
            
            // Evita loop infinito em caso de overflow
            if (address < reinterpret_cast<uintptr_t>(mbi.BaseAddress)) {
                break;
            }
        }

        return regions;
    }
};

// Classe para escanear a memória em busca de valores
template <typename T>
class MemoryScanner {
private:
    MemoryManager& memoryManager;
    std::vector<uintptr_t> results;

public:
    MemoryScanner(MemoryManager& manager) : memoryManager(manager) {}

    // Realiza um primeiro escaneamento em busca de um valor específico
    void firstScan(const T& value) {
        results.clear();
        
        if (!memoryManager.isConnected()) {
            std::cerr << "Não conectado a nenhum processo" << std::endl;
            return;
        }

        std::cout << "Iniciando escaneamento..." << std::endl;
        auto startTime = std::chrono::high_resolution_clock::now();

        auto regions = memoryManager.getMemoryRegions();
        size_t totalBytes = 0;
        size_t scannedBytes = 0;

        // Calcula o total de bytes a serem escaneados
        for (const auto& region : regions) {
            totalBytes += region.second;
        }

        for (const auto& region : regions) {
            uintptr_t start = region.first;
            SIZE_T size = region.second;
            
            // Aloca um buffer para ler a região de memória
            std::vector<BYTE> buffer(size);
            SIZE_T bytesRead;
            
            if (ReadProcessMemory(memoryManager.processHandle, reinterpret_cast<LPCVOID>(start), buffer.data(), size, &bytesRead)) {
                // Escaneia o buffer em busca do valor
                for (SIZE_T i = 0; i <= bytesRead - sizeof(T); i += sizeof(T)) {
                    T* currentValue = reinterpret_cast<T*>(&buffer[i]);
                    if (*currentValue == value) {
                        results.push_back(start + i);
                    }
                }
            }
            
            scannedBytes += size;
            
            // Atualiza o progresso a cada 10%
            static int lastProgress = 0;
            int progress = static_cast<int>((scannedBytes * 100) / totalBytes);
            if (progress >= lastProgress + 10) {
                std::cout << "Progresso: " << progress << "%" << std::endl;
                lastProgress = progress;
            }
        }

        auto endTime = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(endTime - startTime).count();
        
        std::cout << "Escaneamento concluído em " << duration << " ms" << std::endl;
        std::cout << "Resultados encontrados: " << results.size() << std::endl;
    }

    // Realiza um escaneamento subsequente para refinar os resultados
    void nextScan(const T& value) {
        if (results.empty()) {
            std::cerr << "Nenhum resultado para refinar. Faça um primeiro escaneamento." << std::endl;
            return;
        }

        std::cout << "Refinando resultados..." << std::endl;
        auto startTime = std::chrono::high_resolution_clock::now();

        std::vector<uintptr_t> newResults;
        
        for (const auto& address : results) {
            T currentValue;
            if (memoryManager.readMemory(address, currentValue) && currentValue == value) {
                newResults.push_back(address);
            }
        }
        
        results = newResults;
        
        auto endTime = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(endTime - startTime).count();
        
        std::cout << "Refinamento concluído em " << duration << " ms" << std::endl;
        std::cout << "Resultados encontrados: " << results.size() << std::endl;
    }

    // Modifica todos os valores encontrados
    void modifyValues(const T& newValue) {
        if (results.empty()) {
            std::cerr << "Nenhum resultado para modificar." << std::endl;
            return;
        }

        std::cout << "Modificando " << results.size() << " endereços..." << std::endl;
        
        int successCount = 0;
        for (const auto& address : results) {
            if (memoryManager.writeMemory(address, newValue)) {
                successCount++;
            }
        }
        
        std::cout << "Modificação concluída. " << successCount << " endereços modificados com sucesso." << std::endl;
    }

    // Exibe os resultados encontrados
    void showResults(int limit = 20) {
        if (results.empty()) {
            std::cout << "Nenhum resultado encontrado." << std::endl;
            return;
        }

        std::cout << "Resultados encontrados: " << results.size() << std::endl;
        std::cout << "Mostrando até " << limit << " resultados:" << std::endl;
        
        std::cout << std::hex << std::uppercase;
        
        int count = 0;
        for (const auto& address : results) {
            if (count >= limit) break;
            
            T value;
            if (memoryManager.readMemory(address, value)) {
                std::cout << "Endereço: 0x" << address << " | Valor: ";
                
                // Exibe o valor de acordo com o tipo
                if constexpr (std::is_same_v<T, int> || std::is_same_v<T, long> || 
                             std::is_same_v<T, short> || std::is_same_v<T, char>) {
                    std::cout << std::dec << static_cast<int>(value) << std::hex;
                } else if constexpr (std::is_same_v<T, float> || std::is_same_v<T, double>) {
                    std::cout << std::dec << value << std::hex;
                } else {
                    std::cout << value;
                }
                
                std::cout << std::endl;
            }
            
            count++;
        }
        
        std::cout << std::dec << std::nouppercase;
    }

    // Obtém o número de resultados
    size_t getResultCount() const {
        return results.size();
    }

    // Limpa os resultados
    void clearResults() {
        results.clear();
        std::cout << "Resultados limpos." << std::endl;
    }
};

// Função principal
int main() {
    MemoryManager memoryManager;
    
    std::cout << "=== Scanner de Memória Básico ===" << std::endl;
    std::cout << "Este programa permite escanear a memória de um processo em busca de valores específicos." << std::endl;
    std::cout << "AVISO: Use este programa apenas para fins educacionais e em processos que você tem permissão para acessar." << std::endl;
    std::cout << std::endl;

    while (true) {
        std::cout << "\n=== Menu Principal ===" << std::endl;
        std::cout << "1. Listar processos" << std::endl;
        std::cout << "2. Conectar a um processo (por ID)" << std::endl;
        std::cout << "3. Conectar a um processo (por nome)" << std::endl;
        std::cout << "4. Escanear memória (int)" << std::endl;
        std::cout << "5. Escanear memória (float)" << std::endl;
        std::cout << "6. Sair" << std::endl;
        
        if (memoryManager.isConnected()) {
            std::cout << "\nProcesso conectado: " << memoryManager.getProcessName() 
                      << " (ID: " << memoryManager.getProcessId() << ")" << std::endl;
        }
        
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
                std::cout << std::left << std::setw(8) << "ID" << " | " << "Nome" << std::endl;
                std::cout << "------------------------" << std::endl;
                
                auto processes = memoryManager.listProcesses();
                for (const auto& process : processes) {
                    std::cout << std::left << std::setw(8) << process.first << " | " << process.second << std::endl;
                }
                break;
            }
            
            case 2: {
                // Conectar a um processo por ID
                std::cout << "Digite o ID do processo: ";
                DWORD pid;
                std::cin >> pid;
                
                if (std::cin.fail()) {
                    std::cin.clear();
                    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                    std::cout << "ID inválido." << std::endl;
                    break;
                }
                
                if (memoryManager.connectToProcess(pid)) {
                    std::cout << "Conectado ao processo " << memoryManager.getProcessName() 
                              << " (ID: " << memoryManager.getProcessId() << ")" << std::endl;
                }
                break;
            }
            
            c
(Content truncated due to size limit. Use line ranges to read in chunks)