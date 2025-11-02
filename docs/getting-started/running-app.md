import flet as ft
import requests
import json

# --- 1. CONFIGURAÇÃO DA API ---
# !!! ATENÇÃO: SUBSTITUA ESTA CHAVE PELA SUA CHAVE REAL DO OpenWeatherMap !!!
API_KEY = "SUA_CHAVE_DA_API_AQUI"
BASE_URL = "http://api.openweathermap.org/data/2.5/weather"

# --- 2. FUNÇÕES DE BUSCA E LÓGICA ---

def obter_previsao(cidade: str) -> dict | None:
    """Busca a previsão do tempo na API do OpenWeatherMap."""
    
    if not API_KEY or API_KEY == "SUA_CHAVE_DA_API_AQUI":
        # Retorna um erro amigável se a chave não foi configurada
        return {"error": "ERRO: A chave da API não foi configurada."}

    params = {
        'q': cidade,
        'appid': API_KEY,
        'units': 'metric',  # Unidades métricas (Celsius)
        'lang': 'pt_br'     # Linguagem em Português
    }
    
    try:
        response = requests.get(BASE_URL, params=params)
        response.raise_for_status() # Lança exceção para status de erro (4xx ou 5xx)
        dados = response.json()

        if dados.get('cod') == 200:
            return dados
        else:
            return {"error": f"Erro na API: {dados.get('message', 'Erro desconhecido')}"}

    except requests.exceptions.HTTPError as e:
        if response.status_code == 404:
            return {"error": f"Cidade '{cidade}' não encontrada."}
        else:
            return {"error": f"Erro HTTP: {e}"}
            
    except requests.exceptions.RequestException as e:
        return {"error": f"Erro de Conexão: {e}"}

# --- 3. FUNÇÃO PRINCIPAL DO APLICATIVO FLET ---

def main(page: ft.Page):
    page.title = "☀️ Flet Verificador de Previsão do Tempo (SPA)"
    page.vertical_alignment = ft.MainAxisAlignment.CENTER
    page.horizontal_alignment = ft.CrossAxisAlignment.CENTER
    page.theme_mode = ft.ThemeMode.LIGHT
    page.window_width = 400
    page.window_height = 550
    page.update()
    
    # Dicionário de Mapeamento de Ícones do OpenWeatherMap para Ícones do Flet/Material
    # Observação: O Flet usa ícones do Material. Vamos mapear para alguns que se assemelham.
    ICON_MAP = {
        "01d": ft.icons.WB_SUNNY,           # Céu Limpo (Dia)
        "01n": ft.icons.NIGHTLIGHT,         # Céu Limpo (Noite)
        "02d": ft.icons.PARTLY_CLOUDY_DAY,  # Nuvens Esparsas (Dia)
        "02n": ft.icons.PARTLY_CLOUDY_NIGHT,# Nuvens Esparsas (Noite)
        "03d": ft.icons.CLOUDY,             # Nuvens Dispersas
        "03n": ft.icons.CLOUDY,
        "04d": ft.icons.CLOUD,              # Nublado
        "04n": ft.icons.CLOUD,
        "09d": ft.icons.WATER_DROP,         # Chuva Fraca
        "09n": ft.icons.WATER_DROP,
        "10d": ft.icons.RAINY,              # Chuva
        "10n": ft.icons.RAINY,
        "11d": ft.icons.FLASH_ON,           # Trovoadas
        "11n": ft.icons.FLASH_ON,
        "13d": ft.icons.SNOWING,            # Neve
        "13n": ft.icons.SNOWING,
        "50d": ft.icons.AIR,                # Névoa
        "50n": ft.icons.AIR,
    }
    
    # --- 4. ELEMENTOS DE EXIBIÇÃO ---
    
    # Campo de Entrada
    cidade_input = ft.TextField(
        label="Digite o nome da cidade",
        hint_text="Ex: Rio de Janeiro",
        width=300,
        autofocus=True,
        on_submit=lambda e: buscar_clima(e), # Permite buscar ao pressionar Enter
        border_radius=10,
    )

    # Elementos de Resultado
    icone_clima = ft.Icon(name=ft.icons.HELP_OUTLINE, size=80, color=ft.colors.BLUE_GREY_400)
    temperatura_texto = ft.Text("---", size=48, weight=ft.FontWeight.BOLD)
    descricao_texto = ft.Text("Aguardando pesquisa...", size=20, italic=True)
    cidade_nome = ft.Text("Local", size=24, weight=ft.FontWeight.W_600)
    erro_texto = ft.Text("", size=16, color=ft.colors.RED_500)
    
    # Card de Resultado
    resultado_card = ft.Card(
        content=ft.Container(
            content=ft.Column(
                [
                    cidade_nome,
                    ft.Divider(height=10, color=ft.colors.BLUE_GREY_100),
                    icone_clima,
                    temperatura_texto,
                    descricao_texto,
                ],
                horizontal_alignment=ft.CrossAxisAlignment.CENTER,
                spacing=10
            ),
            padding=30,
            width=300,
            alignment=ft.alignment.center,
        ),
        elevation=10,
        visible=False, # Começa invisível
    )

    # --- 5. FUNÇÃO DE AÇÃO (BUSCA) ---

    def buscar_clima(e):
        """Função chamada ao clicar no botão ou pressionar Enter."""
        cidade = cidade_input.value.strip()
        
        if not cidade:
            erro_texto.value = "Por favor, digite o nome de uma cidade."
            resultado_card.visible = False
            page.update()
            return

        # Limpa erros anteriores
        erro_texto.value = ""
        resultado_card.visible = False # Esconde o card enquanto carrega

        # Mostra um indicador de carregamento (simples)
        cidade_nome.value = "Buscando..."
        temperatura_texto.value = ""
        descricao_texto.value = ""
        icone_clima.name = ft.icons.SEARCH
        resultado_card.visible = True
        page.update()
        
        # Chama a função de busca da API
        dados = obter_previsao(cidade)
        
        if dados and "error" in dados:
            # Caso de erro
            erro_texto.value = dados["error"]
            resultado_card.visible = False # Esconde o card de novo
        else:
            # Sucesso
            temperatura = dados['main']['temp']
            descricao = dados['weather'][0]['description'].capitalize()
            icone_code = dados['weather'][0]['icon']
            nome_cidade = dados['name']
            
            # Atualiza os componentes
            cidade_nome.value = nome_cidade
            temperatura_texto.value = f"{temperatura:.1f}°C"
            descricao_texto.value = descricao
            icone_clima.name = ICON_MAP.get(icone_code, ft.icons.CLOUD_OFF)
            icone_clima.color = ft.colors.AMBER_500 if icone_code in ["01d"] else ft.colors.BLUE
            
            # Garante que o card está visível e limpa erros
            resultado_card.visible = True
            erro_texto.value = ""

        page.update() # Atualiza a UI

    # --- 6. MONTAGEM DA INTERFACE (SPA) ---
    
    page.add(
        ft.Container(
            content=ft.Column(
                [
                    ft.Text("Verificador de Previsão do Tempo", size=28, weight=ft.FontWeight.W_800),
                    ft.Row(
                        [
                            cidade_input,
                            ft.IconButton(
                                icon=ft.icons.SEARCH,
                                icon_size=30,
                                on_click=buscar_clima,
                                tooltip="Buscar Clima",
                                style=ft.ButtonStyle(
                                    shape=ft.RoundedRectangleBorder(radius=10)
                                )
                            ),
                        ],
                        alignment=ft.MainAxisAlignment.CENTER,
                        spacing=5
                    ),
                    erro_texto,
                    ft.Container(height=20), # Espaçamento
                    resultado_card,
                ],
                horizontal_alignment=ft.CrossAxisAlignment.CENTER,
                spacing=20
            ),
            padding=30,
            alignment=ft.alignment.center,
        )
    )

if __name__ == "__main__":
    ft.app(target=main)
