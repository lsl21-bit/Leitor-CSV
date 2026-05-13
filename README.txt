#include "StdAfx.h" 
#include <iostream>
#include <string>
#include <windows.h>
#include <wininet.h> 
#include <vector>
#include <sstream>
#include <algorithm> 

#pragma comment(lib, "wininet.lib")

using namespace std;


//ARQUIVO ÚNICO (GERADOR/SOLAR/BESS)

const string TOKEN_INFLUX = "8MTJNqurHp2zK8Cv4hIk97ZFSsw615IJOJ9vnqzfbctnyRlR8qycIy2MVGYpy5AOBQQnujmkZoTkkQThKjawHLtYKfQ_g=="; 
const string ORG_ID = "fa4e574489898160b0af"; 
const string BUCKET_NAME = "Gerador"; 
const string URL_API = "/api/v2/write?org=" + ORG_ID + "&bucket=" + BUCKET_NAME + "&precision=s";

string limpar_path(const string& s) {
    size_t p1 = s.find_first_not_of(" \t\r\n\"");
    if (p1 == string::npos) return "";
    size_t p2 = s.find_last_not_of(" \t\r\n\"");
    return s.substr(p1, (p2 - p1 + 1));
}

vector<string> split(const string& s, char delimitador) {
    vector<string> tokens;
    string token;
    istringstream tokenStream(s);
    while (getline(tokenStream, token, delimitador)) {
        tokens.push_back(token);
    }
    return tokens;
}

string limpar_nome_coluna(string s) {
    replace(s.begin(), s.end(), ' ', '_'); 
    replace(s.begin(), s.end(), '-', '_');
    replace(s.begin(), s.end(), '/', '_');
    s.erase(remove(s.begin(), s.end(), '\"'), s.end());
    return s;
}

string limpar_numero(string s) {
    s.erase(remove(s.begin(), s.end(), '\r'), s.end());
    s.erase(remove(s.begin(), s.end(), '\n'), s.end());
    s.erase(remove(s.begin(), s.end(), ' '), s.end());
    s.erase(remove(s.begin(), s.end(), '\"'), s.end());
    if (s.empty()) return "0.0";
    return s;
}

vector<string> ExtrairCabecalhos(string path) {
    HANDLE hFile = CreateFileA(path.c_str(), GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile == INVALID_HANDLE_VALUE) return vector<string>();
    char buffer[2048];
    DWORD lidos;
    vector<string> cols;
    if (ReadFile(hFile, buffer, sizeof(buffer) - 1, &lidos, NULL)) {
        buffer[lidos] = '\0';
        string s(buffer);
        size_t pos = s.find('\n');
        if (pos != string::npos) {
            cols = split(s.substr(0, pos), ',');
            for(size_t i = 0; i < cols.size(); i++) cols[i] = limpar_nome_coluna(cols[i]);
        }
    }
    CloseHandle(hFile);
    return cols;
}

string LerUltimaLinha(string path) {
    HANDLE hFile = CreateFileA(path.c_str(), GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile == INVALID_HANDLE_VALUE) return "";
    char buffer[2048];
    DWORD lidos;
    string resultado = "";
    LARGE_INTEGER size;
    GetFileSizeEx(hFile, &size);
    LONG offset = (size.QuadPart > 1000) ? -1000 : (LONG)-size.QuadPart;
    SetFilePointer(hFile, offset, NULL, FILE_END);
    if (ReadFile(hFile, buffer, sizeof(buffer) - 1, &lidos, NULL)) {
        buffer[lidos] = '\0';
        string s(buffer);
        size_t n1 = s.find_last_of('\n');
        if (n1 != string::npos) {
            size_t n2 = s.find_last_of('\n', n1 - 1);
            if (n2 != string::npos) resultado = s.substr(n2 + 1, n1 - n2 - 1);
        }
    }
    CloseHandle(hFile);
    return resultado;
}
//COMUNICAÇÃO INFLUXDB
void EnviarParaInflux(string payload) {
    HINTERNET hSess = InternetOpenA("Britacal_Core", INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, 0);
    HINTERNET hConn = InternetConnectA(hSess, "127.0.0.1", 8086, NULL, NULL, INTERNET_SERVICE_HTTP, 0, 1);
    HINTERNET hReq = HttpOpenRequestA(hConn, "POST", URL_API.c_str(), NULL, NULL, NULL, 0, 1);
    string head = "Authorization: Token " + TOKEN_INFLUX + "\r\nContent-Type: text/plain\r\n";
    if (HttpSendRequestA(hReq, head.c_str(), (DWORD)head.length(), (LPVOID)payload.c_str(), (DWORD)payload.length())) {
        DWORD code = 0, len = sizeof(code);
        HttpQueryInfoA(hReq, HTTP_QUERY_STATUS_CODE | HTTP_QUERY_FLAG_NUMBER, &code, &len, NULL);
        if (code == 204) cout << "[OK] Dados Britacal Enviados!" << endl;
    }
    InternetCloseHandle(hReq); 
    InternetCloseHandle(hConn); 
    InternetCloseHandle(hSess);
}

int main() {
    cout << "*******************************************" << endl;
    cout << "*   SUPERVISORIO BRITACAL -	CSV UNICO     *" << endl;
    cout << "*******************************************" << endl;
    
    string caminho;
    cout << "Arraste o arquivo CSV  e pressione Enter: ";
    getline(cin, caminho);
    caminho = limpar_path(caminho);
    
    vector<string> cabecalhos = ExtrairCabecalhos(caminho);
    if(cabecalhos.empty()) { 
        cout << "Erro ao ler arquivo!"; 
        Sleep(3000); 
        return 0; 
    }

    cout << "Monitorando " << cabecalhos.size() << " parametros..." << endl;
    string ultima_l = "";

    while(true) {
        string linha = LerUltimaLinha(caminho);
        if(!linha.empty() && linha != ultima_l && linha.find("Timestamp") == string::npos) {
            vector<string> colunas = split(linha, ',');
            if(colunas.size() > 1) {
                stringstream ss;
                ss << "deif_data,maquina=Microgrid_Britacal "; 
                
                bool primeiro = true;
                for(size_t i = 1; i < colunas.size(); i++) { 
                    string val = limpar_numero(colunas[i]);
                    if(!val.empty()) {
                        if(!primeiro) ss << ",";
                        
                        string nome;
                        if (i < cabecalhos.size()) {
                            nome = cabecalhos[i];
                        } else {
                            stringstream ss_nome;
                            ss_nome << "sensor_" << i;
                            nome = ss_nome.str();
                        }
                        
                        ss << nome << "=" << val;
                        primeiro = false;
                    }
                }
                EnviarParaInflux(ss.str());
                ultima_l = linha;
            }
        }
        Sleep(1000);
    }
    return 0;
}
