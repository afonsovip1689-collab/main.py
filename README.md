import os
import re
import urllib.request
import sqlite3

# Configurações do Projeto
URL_EVENTO = "https://eventos.ifgoiano.edu.br/integra2026/"
ARQUIVO_TXT = "pagina_fonte.html"
PASTA_DOWNLOAD = "downloads"
BANCO_DADOS = "event.db"

def tarefa_002_baixar_html():
    """Baixa o código-fonte da página do evento e salva em um arquivo de texto."""
    print("[Tarefa 002] Baixando código-fonte da página...")
    headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
    requisicao = urllib.request.Request(URL_EVENTO, headers=headers)
    
    with urllib.request.urlopen(requisicao) as resposta:
        conteudo_html = resposta.read().decode('utf-8')
    
    with open(ARQUIVO_TXT, "w", encoding="utf-8") as f:
        f.write(conteudo_html)
    
    print(f"Código-fonte salvo em: {ARQUIVO_TXT}")
    return conteudo_html

def tarefa_003_extrair_dados_regex(html_content):
    """Aplica expressões regulares para extrair os dados dos palestrantes."""
    print("[Tarefa 003] Extraindo dados via REGEX...")
    
    padrao = re.compile(
        r'<img\s+src=["\'](?P<imagem>[^"\']+)["\'][^>]*class=["\'][^"\']*img-palestrante[^"\']*["\'][^>]*>.*?'
        r'<h3[^>]*>(?P<nome>[^<]+)</h3>.*?'
        r'<h5[^>]*>(?P<trabalho>[^<]+)</h5>.*?'
        r'<p[^>]*>(?P<email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})</p>',
        re.DOTALL | re.IGNORECASE
    )

    palestrantes = []
    for match in padrao.finditer(html_content):
        dados = match.groupdict()
        palestrantes.append({
            'imagem': dados['imagem'].strip(),
            'nome': dados['nome'].strip(),
            'trabalho': dados['trabalho'].strip(),
            'email': dados['email'].strip()
        })
    
    print(f"Total de palestrantes encontrados: {len(palestrantes)}")
    return palestrantes

def tarefa_004_baixar_imagens(palestrantes):
    """Baixa as imagens dos palestrantes na pasta download."""
    print("[Tarefa 004] Baixando imagens dos palestrantes...")
    if not os.path.exists(PASTA_DOWNLOAD):
        os.makedirs(PASTA_DOWNLOAD)

    for i, p in enumerate(palestrantes):
        url_imagem = p['imagem']
        if not url_imagem.startswith('http'):
            url_imagem = urllib.parse.urljoin(URL_EVENTO, url_imagem)

        nome_arquivo = os.path.basename(urllib.parse.urlparse(url_imagem).path)
        if not nome_arquivo:
            nome_arquivo = f"palestrante_{i+1}.jpg"

        caminho_local = os.path.join(PASTA_DOWNLOAD, nome_arquivo)
        
        try:
            urllib.request.urlretrieve(url_imagem, caminho_local)
            p['nome_imagem_local'] = nome_arquivo
        except Exception as e:
            print(f"Erro ao baixar imagem de {p['nome']}: {e}")
            p['nome_imagem_local'] = nome_arquivo

def tarefa_005_criar_banco():
    """Cria a tabela speaker no banco de dados SQLite event.db."""
    print("[Tarefa 005] Configurando o Banco de Dados SQLite...")
    conn = sqlite3.connect(BANCO_DADOS)
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS speaker (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name VARCHAR(255) NOT NULL,
            work VARCHAR(255) NOT NULL,
            email VARCHAR(255) NOT NULL,
            image VARCHAR(255) NOT NULL
        )
    ''')
    
    conn.commit()
    conn.close()

def tarefa_006_inserir_dados(palestrantes):
    """Registra os palestrantes extraídos na tabela speaker."""
    print("[Tarefa 006] Inserindo dados no banco de dados...")
    conn = sqlite3.connect(BANCO_DADOS)
    cursor = conn.cursor()

    for p in palestrantes:
        cursor.execute('''
            INSERT INTO speaker (name, work, email, image)
            VALUES (?, ?, ?, ?)
        ''', (p['nome'], p['trabalho'], p['email'], p['nome_imagem_local']))

    conn.commit()
    conn.close()
    print("Processo finalizado com sucesso!")

if __name__ == "__main__":
    conteudo_html = tarefa_002_baixar_html()
    palestrantes = tarefa_003_extrair_dados_regex(conteudo_html)
    
    if palestrantes:
        tarefa_004_baixar_imagens(palestrantes)
        tarefa_005_criar_banco()
        tarefa_006_inserir_dados(palestrantes)
    else:
        print("Nenhum palestrante foi identificado via REGEX.")
