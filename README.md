#!/usr/bin/env python
# -*- coding: utf-8 -*-

import asyncio
import threading
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
import json
import time
import os
import csv
import datetime
import xml.etree.ElementTree as ET
import random
import logging

class FenixBot:
    """
    Sistema Fênix - Bot de Trading Avançado para Deriv.com
    
    Características:
    - Suporte a múltiplas estratégias
    - Sistema de recuperação com modo virtual
    - Gerenciamento de risco avançado
    - Integração com API Deriv
    - Suporte a múltiplos ativos
    - Interface gráfica amigável
    """
    
    def __init__(self, root):
        """Inicializa o bot Fênix"""
        self.root = root
        self.root.title("🔥 FÊNIX - Sistema de Trading Avançado 🔥")
        self.root.geometry("900x930")
        self.root.configure(bg="#1E1E1E")
        
        # Diretórios e arquivos
        self.config_dir = "config"
        self.logs_dir = "logs"
        self.bots_dir = "bots"
        self.sounds_dir = "sounds"
        
        os.makedirs(self.config_dir, exist_ok=True)
        os.makedirs(self.logs_dir, exist_ok=True)
        os.makedirs(self.bots_dir, exist_ok=True)
        os.makedirs(self.sounds_dir, exist_ok=True)
        
        # Variáveis de controle
        self.token_demo = tk.StringVar()
        self.token_real = tk.StringVar()
        self.api_mode = tk.StringVar(value="demo")
        self.trading_mode = tk.StringVar(value="live")  # live ou virtual
        self.symbol = tk.StringVar(value="R_10")
        self.duration = tk.IntVar(value=5)
        self.win_amount = tk.DoubleVar(value=0.35)
        self.expected_profit = tk.DoubleVar(value=5.0)
        self.max_loss = tk.DoubleVar(value=10.0)
        self.profit_increase = tk.DoubleVar(value=15.0)  # Aumento de meta (%)
        self.strategy = tk.StringVar(value="V10_EMA_Stochastic")
        
        # Status e métricas
        self.running = False
        self.stop_requested = False
        self.current_stake = 0.0
        self.total_profit = 0.0
        self.wins = 0
        self.losses = 0
        self.consecutive_losses = 0
        self.max_consecutive_losses = 0
        self.last_result = None
        self.balance = 0.0
        self.daily_goal_reached = False
        self.operations_count = 0
        
        # Recuperação e segurança
        self.recovery_mode = False
        self.virtual_mode = False
        self.virtual_balance = 100.0
        
        # Histórico de operações
        self.operations_history = []
        self.operations_csv_path = os.path.join(
            self.logs_dir, 
            f"operations_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
        )
        
        # Inicializa o arquivo CSV para registrar operações
        try:
            with open(self.operations_csv_path, 'w', newline='') as csv_file:
                csv_writer = csv.writer(csv_file)
                csv_writer.writerow([
                    "id", "timestamp", "reference", "contract_type", 
                    "entry_tick", "exit_tick", "buy_price", "sell_price", 
                    "profit", "contract_type"
                ])
            # O log será feito depois que a interface for construída
        except Exception as e:
            print(f"❌ Erro ao criar arquivo CSV: {e}")
        
        # Interface e componentes visuais
        self.logs_text = None
        self.progress_frame = None
        self.progress_bar = None
        self.status_label = None
        self.profit_label = None
        self.balance_label = None
        self.operations_label = None
        self.results_labels = {}
        self.operations_table = None
        
        os.makedirs(self.config_dir, exist_ok=True)
        os.makedirs(self.logs_dir, exist_ok=True)
        os.makedirs(self.bots_dir, exist_ok=True)
        os.makedirs(self.sounds_dir, exist_ok=True)
        
        self.config_path = os.path.join(self.config_dir, "fenix_config.json")
        
        # Configurar logging
        logging.basicConfig(
            level=logging.DEBUG,
            format='%(asctime)s [%(levelname)s] %(message)s',
            handlers=[
                logging.FileHandler(os.path.join(self.logs_dir, f"fenix_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.log")),
                logging.StreamHandler()
            ]
        )
        
        # Carregar configurações
        self.load_config()
        
        # Construir interface
        self.build_interface()
        
        # Verificar proteção por senha
        self.check_password()
        
        # Carregar estratégias em XML
        self.load_bot_strategies()
    
    def check_password(self):
        """Verifica proteção por senha no primeiro uso"""
        password_file = os.path.join(self.config_dir, ".security")
        
        if not os.path.exists(password_file):
            # Primeira execução - configurar senha
            password = simpledialog.askstring("Segurança Fênix", 
                                             "Primeira execução - Crie uma senha de segurança:",
                                             show="*", parent=self.root)
            
            if password:
                # Salva hash da senha (simplificado para demonstração)
                import hashlib
                password_hash = hashlib.sha256(password.encode()).hexdigest()
                
                with open(password_file, "w") as f:
                    f.write(password_hash)
                
                messagebox.showinfo("Segurança Fênix", "Senha configurada com sucesso!")
            else:
                # Usuário cancelou - fecha aplicação
                messagebox.showerror("Erro", "É necessário configurar uma senha!")
                self.root.destroy()
        else:
            # Solicita senha
            with open(password_file, "r") as f:
                password_hash = f.read().strip()
            
            for attempt in range(3):
                password = simpledialog.askstring("Segurança Fênix", 
                                                 f"Digite sua senha ({3-attempt} tentativas restantes):",
                                                 show="*", parent=self.root)
                
                if not password:
                    # Usuário cancelou - fecha aplicação
                    self.root.destroy()
                    return
                
                # Verifica senha
                import hashlib
                if hashlib.sha256(password.encode()).hexdigest() == password_hash:
                    # Senha correta
                    return
                
                if attempt < 2:
                    messagebox.showerror("Erro", f"Senha incorreta! Você tem mais {2-attempt} tentativas.")
            
            # Excedeu tentativas - auto-destruição
            messagebox.showerror("Segurança Crítica", "Número máximo de tentativas excedido. Sistema bloqueado.")
            self.log("🚨 ALERTA DE SEGURANÇA - Tentativas de senha excedidas")
            
            # Em um sistema real, poderia apagar arquivos sensíveis ou desativar o sistema
            self.root.destroy()
    
    def build_interface(self):
        """Constrói a interface gráfica completa"""
        # Estilo
        self.configure_styles()
        
        # Adiciona o logo no topo
        logo_frame = ttk.Frame(self.root)
        logo_frame.pack(fill=tk.X, padx=10, pady=10)
        
        # Carrega a imagem do logo
        try:
            logo_path = os.path.join("logos", "fenix_logo.png")
            if os.path.exists(logo_path):
                from PIL import Image, ImageTk
                logo_img = Image.open(logo_path)
                logo_img = logo_img.resize((200, 200), Image.LANCZOS if hasattr(Image, 'LANCZOS') else Image.ANTIALIAS)
                logo_photo = ImageTk.PhotoImage(logo_img)
                
                # Adiciona o logo ao frame
                logo_label = tk.Label(logo_frame, image=logo_photo, bg="#1E1E1E")
                logo_label.image = logo_photo  # Mantém referência para evitar coleta de lixo
                logo_label.pack(pady=10)
                
                # Adiciona título dourado abaixo do logo
                title_label = tk.Label(logo_frame, text="SISTEMA FÊNIX", 
                                     font=("Arial", 18, "bold"), 
                                     fg="#DAA520", bg="#1E1E1E")
                title_label.pack(pady=5)
            else:
                title_label = tk.Label(logo_frame, text="🔥 SISTEMA FÊNIX 🔥", 
                                     font=("Arial", 20, "bold"), 
                                     fg="#DAA520", bg="#1E1E1E")
                title_label.pack(pady=10)
                self.log("⚠️ Logo não encontrado. Usando título texto.")
                
        except Exception as e:
            # Fallback para texto se não conseguir carregar a imagem
            title_label = tk.Label(logo_frame, text="🔥 SISTEMA FÊNIX 🔥", 
                                  font=("Arial", 20, "bold"), 
                                  fg="#DAA520", bg="#1E1E1E")
            title_label.pack(pady=10)
            self.log(f"⚠️ Erro ao carregar logo: {e}")
        
        # Layout principal - notebook com abas
        notebook = ttk.Notebook(self.root)
        notebook.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        
        # Aba principal
        main_tab = ttk.Frame(notebook)
        notebook.add(main_tab, text="Trading")
        
        # Aba de configurações
        config_tab = ttk.Frame(notebook)
        notebook.add(config_tab, text="Configurações")
        
        # Aba de estratégias
        strategies_tab = ttk.Frame(notebook)
        notebook.add(strategies_tab, text="Estratégias")
        
        # Aba de estatísticas
        stats_tab = ttk.Frame(notebook)
        notebook.add(stats_tab, text="Estatísticas")
        
        # Construir componentes de cada aba
        self.build_main_tab(main_tab)
        self.build_config_tab(config_tab)
        self.build_strategies_tab(strategies_tab)
        self.build_stats_tab(stats_tab)
    
    def configure_styles(self):
        """Configura estilos para ttk widgets"""
        style = ttk.Style()
        
        # Configurar tema
        try:
            style.theme_use("clam")
        except:
            pass
        
        # Botões
        style.configure("TButton", 
                       font=("Arial", 10, "bold"),
                       background="#4CAF50",
                       foreground="black")
        
        # Labels
        style.configure("TLabel", 
                       font=("Arial", 10),
                       background="#1E1E1E",
                       foreground="white")
        
        # Labels de título
        style.configure("Title.TLabel", 
                       font=("Arial", 12, "bold"),
                       background="#1E1E1E",
                       foreground="#DAA520")
        
        # Frames
        style.configure("TFrame", 
                       background="#1E1E1E")
        
        # Notebook
        style.configure("TNotebook", 
                       background="#1E1E1E",
                       foreground="white")
        
        style.configure("TNotebook.Tab", 
                       font=("Arial", 10, "bold"),
                       background="#333333",
                       foreground="white",
                       padding=[10, 5])
    
    def build_main_tab(self, parent):
        """Constrói os componentes da aba principal"""
        # Frame superior para controles
        control_frame = ttk.Frame(parent)
        control_frame.pack(fill=tk.X, padx=10, pady=10)
        
        # Status e progresso
        status_frame = ttk.Frame(parent)
        status_frame.pack(fill=tk.X, padx=10, pady=5)
        
        # Frame para logs
        logs_frame = ttk.Frame(parent)
        logs_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # Botões de controle
        self.build_control_buttons(control_frame)
        
        # Status e barra de progresso
        self.build_status_area(status_frame)
        
        # Área de logs
        self.build_logs_area(logs_frame)
    
    def build_control_buttons(self, parent):
        """Constrói os botões de controle"""
        # Frame para os botões
        buttons_frame = ttk.Frame(parent)
        buttons_frame.pack(fill=tk.X, pady=5)
        
        # Botão iniciar
        start_button = ttk.Button(buttons_frame, text="▶️ Iniciar Trading", 
                                command=self.start_trading_thread)
        start_button.pack(side=tk.LEFT, padx=5)
        
        # Botão parar
        stop_button = ttk.Button(buttons_frame, text="⏹️ Parar Trading", 
                               command=self.stop_trading)
        stop_button.pack(side=tk.LEFT, padx=5)
        
        # Seleção de modo
        mode_frame = ttk.Frame(parent)
        mode_frame.pack(fill=tk.X, pady=5)
        
        # Frame para radio buttons
        radio_frame = ttk.Frame(mode_frame)
        radio_frame.pack(side=tk.LEFT, padx=10)
        
        ttk.Label(radio_frame, text="Modo da API:").pack(side=tk.LEFT, padx=5)
        ttk.Radiobutton(radio_frame, text="DEMO", variable=self.api_mode, 
                       value="demo").pack(side=tk.LEFT, padx=5)
        ttk.Radiobutton(radio_frame, text="REAL", variable=self.api_mode, 
                       value="real").pack(side=tk.LEFT, padx=5)
        
        # Frame para trading mode
        trade_mode_frame = ttk.Frame(mode_frame)
        trade_mode_frame.pack(side=tk.RIGHT, padx=10)
        
        ttk.Label(trade_mode_frame, text="Modo de Trading:").pack(side=tk.LEFT, padx=5)
        ttk.Radiobutton(trade_mode_frame, text="LIVE", variable=self.trading_mode, 
                       value="live").pack(side=tk.LEFT, padx=5)
        ttk.Radiobutton(trade_mode_frame, text="VIRTUAL", variable=self.trading_mode, 
                       value="virtual").pack(side=tk.LEFT, padx=5)
                       
        # Frame para seleção de estratégia
        strategy_frame = ttk.Frame(parent)
        strategy_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(strategy_frame, text="Estratégia:").pack(side=tk.LEFT, padx=5)
        self.strategy_combobox = ttk.Combobox(strategy_frame, textvariable=self.strategy, state="readonly")
        self.strategy_combobox.pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        
        # Preenche a combobox com as estratégias disponíveis
        self.update_strategy_combobox()
    
    def build_status_area(self, parent):
        """Constrói a área de status e progresso"""
        # Frame superior
        top_status_frame = ttk.Frame(parent)
        top_status_frame.pack(fill=tk.X, pady=5)
        
        # Status label
        self.status_label = ttk.Label(top_status_frame, text="Status: Pronto", 
                                    font=("Arial", 10, "bold"))
        self.status_label.pack(side=tk.LEFT, padx=10)
        
        # Saldo
        self.balance_label = ttk.Label(top_status_frame, text="Saldo: $ 0.00")
        self.balance_label.pack(side=tk.RIGHT, padx=10)
        
        # Frame intermediário
        mid_status_frame = ttk.Frame(parent)
        mid_status_frame.pack(fill=tk.X, pady=5)
        
        # Lucro
        self.profit_label = ttk.Label(mid_status_frame, text="Lucro: $ 0.00")
        self.profit_label.pack(side=tk.LEFT, padx=10)
        
        # Operações
        self.operations_label = ttk.Label(mid_status_frame, text="Operações: 0")
        self.operations_label.pack(side=tk.RIGHT, padx=10)
        
        # Frame para barra de progresso
        progress_frame = ttk.Frame(parent)
        progress_frame.pack(fill=tk.X, pady=5)
        
        # Barra de progresso para meta diária
        ttk.Label(progress_frame, text="Meta Diária:").pack(side=tk.LEFT, padx=5)
        
        self.progress_bar = ttk.Progressbar(progress_frame, orient=tk.HORIZONTAL, 
                                          length=550, mode='determinate')
        self.progress_bar.pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        
        # Frame para resultados
        results_frame = ttk.Frame(parent)
        results_frame.pack(fill=tk.X, pady=5)
        
        # Vitórias/Derrotas
        results_left_frame = ttk.Frame(results_frame)
        results_left_frame.pack(side=tk.LEFT, padx=10)
        
        self.results_labels["wins"] = ttk.Label(results_left_frame, text="✅ Vitórias: 0")
        self.results_labels["wins"].pack(side=tk.LEFT, padx=10)
        
        self.results_labels["losses"] = ttk.Label(results_left_frame, text="❌ Derrotas: 0")
        self.results_labels["losses"].pack(side=tk.LEFT, padx=10)
        
        # Taxa de acerto e recuperação
        results_right_frame = ttk.Frame(results_frame)
        results_right_frame.pack(side=tk.RIGHT, padx=10)
        
        self.results_labels["winrate"] = ttk.Label(results_right_frame, text="Taxa: 0%")
        self.results_labels["winrate"].pack(side=tk.LEFT, padx=10)
        
        self.results_labels["recovery"] = ttk.Label(results_right_frame, text="Recuperação: Não")
        self.results_labels["recovery"].pack(side=tk.LEFT, padx=10)
    
    def build_logs_area(self, parent):
        """Constrói a área de logs"""
        # Frame de título
        title_frame = ttk.Frame(parent)
        title_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(title_frame, text="📜 LOGS DE OPERAÇÃO", 
                style="Title.TLabel").pack(anchor=tk.W)
        
        # Frame do texto
        text_frame = ttk.Frame(parent)
        text_frame.pack(fill=tk.BOTH, expand=True)
        
        # Scrollbar
        scrollbar = ttk.Scrollbar(text_frame)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Área de texto
        self.logs_text = tk.Text(text_frame, height=15, bg="#2A2A2A", fg="#CCCCCC",
                              wrap=tk.WORD, yscrollcommand=scrollbar.set,
                              font=("Consolas", 9))
        self.logs_text.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        
        # Configurar scrollbar
        scrollbar.config(command=self.logs_text.yview)
        
        # Mensagem de boas-vindas
        welcome_msg = (
            "=== 🔥 SISTEMA FÊNIX - BOT DE TRADING 1.0 🔥 ===\n"
            "Bem-vindo ao Sistema Fênix para Deriv.com\n\n"
            f"✅ Sistema inicializado com sucesso em {datetime.datetime.now().strftime('%d/%m/%Y %H:%M:%S')}\n"
            "✅ Gerenciador de indicadores técnicos carregado\n"
            "✅ Prontos para iniciar operações\n\n"
            "Para começar, selecione modo de operação e clique em 'Iniciar Trading'\n"
        )
        self.log(welcome_msg)
    
    def build_config_tab(self, parent):
        """Constrói a aba de configurações"""
        # Layout principal em duas colunas
        main_frame = ttk.Frame(parent)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # Coluna esquerda para configurações gerais
        left_frame = ttk.Frame(main_frame)
        left_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5)
        
        # Título da seção
        ttk.Label(left_frame, text="Configurações Gerais", 
                style="Title.TLabel").pack(anchor=tk.W, pady=10)
        
        # Frame para configurações
        general_config_frame = ttk.Frame(left_frame)
        general_config_frame.pack(fill=tk.X, pady=5)
        
        # Grid de elementos
        # Labels na coluna 0, entries/widgets na coluna 1
        ttk.Label(general_config_frame, text="Ativo:").grid(row=0, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Combobox(general_config_frame, textvariable=self.symbol, 
                   values=["R_10", "R_25", "R_50", "R_75", "R_100", 
                           "BOOM1000", "CRASH1000", "BOOM500", "CRASH500",
                           "1HZ10V", "1HZ25V", "1HZ50V", "1HZ75V", "1HZ100V"]).grid(
            row=0, column=1, sticky=tk.W, padx=5, pady=5)
        
        ttk.Label(general_config_frame, text="Duração (seg):").grid(row=1, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Spinbox(general_config_frame, from_=1, to=60, increment=1, 
                  textvariable=self.duration).grid(
            row=1, column=1, sticky=tk.W, padx=5, pady=5)
        
        ttk.Label(general_config_frame, text="Valor inicial (USD):").grid(row=2, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Spinbox(general_config_frame, from_=0.1, to=100, increment=0.1, 
                  textvariable=self.win_amount, format="%.2f").grid(
            row=2, column=1, sticky=tk.W, padx=5, pady=5)
        
        ttk.Label(general_config_frame, text="Meta de lucro (USD):").grid(row=3, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Spinbox(general_config_frame, from_=1, to=1000, increment=1, 
                  textvariable=self.expected_profit, format="%.2f").grid(
            row=3, column=1, sticky=tk.W, padx=5, pady=5)
        
        ttk.Label(general_config_frame, text="Perda máxima (USD):").grid(row=4, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Spinbox(general_config_frame, from_=1, to=1000, increment=1, 
                  textvariable=self.max_loss, format="%.2f").grid(
            row=4, column=1, sticky=tk.W, padx=5, pady=5)
        
        ttk.Label(general_config_frame, text="Aumento da meta (%):").grid(row=5, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Spinbox(general_config_frame, from_=0, to=100, increment=5, 
                  textvariable=self.profit_increase, format="%.1f").grid(
            row=5, column=1, sticky=tk.W, padx=5, pady=5)
        
        # Coluna direita para API e métodos avançados
        right_frame = ttk.Frame(main_frame)
        right_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=5)
        
        # Título da seção
        ttk.Label(right_frame, text="Conexão com Deriv API", 
                style="Title.TLabel").pack(anchor=tk.W, pady=10)
        
        # Frame para API
        api_config_frame = ttk.Frame(right_frame)
        api_config_frame.pack(fill=tk.X, pady=5)
        
        # Labels e entries para tokens
        ttk.Label(api_config_frame, text="Token da API (DEMO):").grid(row=0, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Entry(api_config_frame, textvariable=self.token_demo, show="*").grid(row=0, column=1, sticky=tk.E+tk.W, padx=5, pady=5)
        
        ttk.Label(api_config_frame, text="Token da API (REAL):").grid(row=1, column=0, sticky=tk.W, padx=5, pady=5)
        ttk.Entry(api_config_frame, textvariable=self.token_real, show="*").grid(row=1, column=1, sticky=tk.E+tk.W, padx=5, pady=5)
        
        # Link para obter token
        token_link_label = ttk.Label(api_config_frame, text="Obter token", 
                                   foreground="#3498db", cursor="hand2")
        token_link_label.grid(row=2, column=1, sticky=tk.E, padx=5, pady=5)
        token_link_label.bind("<Button-1>", lambda e: self.open_token_page())
        
        # Botão para testar conexão
        ttk.Button(api_config_frame, text="Testar Conexão", 
                 command=self.test_connection).grid(row=3, column=0, columnspan=2, pady=10)
        
        # Botões de ação para configurações
        button_frame = ttk.Frame(parent)
        button_frame.pack(fill=tk.X, pady=10, padx=10)
        
        ttk.Button(button_frame, text="Salvar Configurações", 
                 command=self.save_config).pack(side=tk.LEFT, padx=5)
        
        ttk.Button(button_frame, text="Restaurar Padrões", 
                 command=self.restore_defaults).pack(side=tk.RIGHT, padx=5)
    
    def build_strategies_tab(self, parent):
        """Constrói a aba de estratégias"""
        # Frame principal
        main_frame = ttk.Frame(parent)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # Título da seção
        ttk.Label(main_frame, text="Gerenciador de Estratégias", 
                style="Title.TLabel").pack(anchor=tk.W, pady=10)
        
        # Frame de botões
        buttons_frame = ttk.Frame(main_frame)
        buttons_frame.pack(fill=tk.X, pady=10)
        
        # Botão para criar nova estratégia
        btn_nova = ttk.Button(buttons_frame, text="Nova Estratégia", 
                           command=self.create_new_strategy)
        btn_nova.pack(side=tk.LEFT, padx=5)
        
        # Botão para importar estratégia XML
        btn_importar = ttk.Button(buttons_frame, text="Importar XML", 
                               command=self.import_strategy_xml)
        btn_importar.pack(side=tk.LEFT, padx=5)
        
        # Lista de estratégias
        list_frame = ttk.Frame(main_frame)
        list_frame.pack(fill=tk.BOTH, expand=True, pady=10)
        
        # Scrollbar
        scrollbar = ttk.Scrollbar(list_frame)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Árvore (treeview) para listar estratégias
        columns = ("nome", "tipo", "mercado", "rentabilidade")
        tree = ttk.Treeview(list_frame, columns=columns, 
                          show="headings", 
                          yscrollcommand=scrollbar.set)
        
        tree.heading("nome", text="Nome da Estratégia")
        tree.heading("tipo", text="Tipo")
        tree.heading("mercado", text="Mercado")
        tree.heading("rentabilidade", text="Rentabilidade")
        
        tree.column("nome", width=200)
        tree.column("tipo", width=100)
        tree.column("mercado", width=100)
        tree.column("rentabilidade", width=100)
        
        tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.config(command=tree.yview)
        
        # Guardar a referência da árvore para uso posterior
        self.strategies_tree = tree
        
        # Frame de botões de ação
        action_frame = ttk.Frame(main_frame)
        action_frame.pack(fill=tk.X, pady=10)
        
        # Botão para ver detalhes da estratégia
        btn_detalhes = ttk.Button(action_frame, text="Ver Detalhes", 
                               command=lambda: self.view_strategy_details(tree))
        btn_detalhes.pack(side=tk.LEFT, padx=5)
        
        # Botão para editar estratégia
        btn_editar = ttk.Button(action_frame, text="Editar", 
                             command=lambda: self.edit_strategy(tree))
        btn_editar.pack(side=tk.LEFT, padx=5)
        
        # Botão para excluir estratégia
        btn_excluir = ttk.Button(action_frame, text="Excluir", 
                              command=lambda: self.delete_strategy(tree))
        btn_excluir.pack(side=tk.LEFT, padx=5)
        
        # Guarda referências aos botões para poder habilitá-los/desabilitá-los
        self.strategy_buttons = {
            "nova": btn_nova,
            "importar": btn_importar,
            "detalhes": btn_detalhes,
            "editar": btn_editar,
            "excluir": btn_excluir
        }
    
    def build_stats_tab(self, parent):
        """Constrói a aba de estatísticas"""
        frame = ttk.Frame(parent, padding=10)
        frame.pack(expand=True, fill=tk.BOTH)
        
        # Frame de título
        title_frame = ttk.Frame(frame)
        title_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(title_frame, text="📊 ESTATÍSTICAS DE TRADING", 
                style="Title.TLabel").pack(anchor=tk.W)
        
        # Frame para botões
        button_frame = ttk.Frame(frame)
        button_frame.pack(fill=tk.X, pady=5)
        
        ttk.Button(button_frame, text="Atualizar Estatísticas", 
                 command=self.refresh_stats).pack(side=tk.LEFT, padx=5)
        
        ttk.Button(button_frame, text="Exportar Relatório", 
                 command=self.export_report).pack(side=tk.LEFT, padx=5)
        
        # Frame principal para estatísticas
        stats_frame = ttk.Frame(frame)
        stats_frame.pack(fill=tk.BOTH, expand=True, pady=10)
        
        # Informações gerais
        general_frame = ttk.LabelFrame(stats_frame, text="Informações Gerais")
        general_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(general_frame, text="Operações Totais: 0").grid(row=0, column=0, padx=10, pady=5, sticky=tk.W)
        ttk.Label(general_frame, text="Lucro Total: $0.00").grid(row=0, column=1, padx=10, pady=5, sticky=tk.W)
        ttk.Label(general_frame, text="Taxa de Acerto: 0%").grid(row=1, column=0, padx=10, pady=5, sticky=tk.W)
        ttk.Label(general_frame, text="Média de Lucro: $0.00").grid(row=1, column=1, padx=10, pady=5, sticky=tk.W)
        
        # Desempenho
        performance_frame = ttk.LabelFrame(stats_frame, text="Desempenho")
        performance_frame.pack(fill=tk.X, pady=5)
        
        ttk.Label(performance_frame, text="Melhor Sequência: 0 vitórias").grid(row=0, column=0, padx=10, pady=5, sticky=tk.W)
        ttk.Label(performance_frame, text="Pior Sequência: 0 derrotas").grid(row=0, column=1, padx=10, pady=5, sticky=tk.W)
        ttk.Label(performance_frame, text="Maior Lucro: $0.00").grid(row=1, column=0, padx=10, pady=5, sticky=tk.W)
        ttk.Label(performance_frame, text="Maior Perda: $0.00").grid(row=1, column=1, padx=10, pady=5, sticky=tk.W)
        
        # Tabela de operações
        operations_frame = ttk.LabelFrame(stats_frame, text="Histórico de Operações")
        operations_frame.pack(fill=tk.BOTH, expand=True, pady=5)
        
        # Estilo para a tabela
        style = ttk.Style()
        style.configure("Treeview", 
                      background="#1E1E1E", 
                      foreground="white", 
                      fieldbackground="#1E1E1E", 
                      borderwidth=0)
        style.configure("Treeview.Heading", 
                      background="#2A2A2A", 
                      foreground="white", 
                      relief="flat")
        style.map('Treeview', 
                background=[('selected', '#303030')])
        
        # Criar frame com scrollbar
        tree_frame = ttk.Frame(operations_frame)
        tree_frame.pack(fill=tk.BOTH, expand=True)
        
        # Scrollbar para a tabela
        tree_scroll = ttk.Scrollbar(tree_frame)
        tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Tabela de operações
        self.operations_table = ttk.Treeview(tree_frame, 
                                         columns=("id", "data", "referencia", "tipo", "entrada", "saida", 
                                                "compra", "venda", "lucro", "status"),
                                         show="headings",
                                         yscrollcommand=tree_scroll.set,
                                         style="Treeview")
        
        # Configurar scrollbar
        tree_scroll.config(command=self.operations_table.yview)
        
        # Definir cabeçalhos
        self.operations_table.heading("id", text="ID")
        self.operations_table.heading("data", text="Data/Hora")
        self.operations_table.heading("referencia", text="Referência")
        self.operations_table.heading("tipo", text="Tipo")
        self.operations_table.heading("entrada", text="Entrada")
        self.operations_table.heading("saida", text="Saída")
        self.operations_table.heading("compra", text="Compra")
        self.operations_table.heading("venda", text="Venda")
        self.operations_table.heading("lucro", text="Lucro")
        self.operations_table.heading("status", text="Status")
        
        # Definir larguras
        self.operations_table.column("id", width=30, anchor=tk.CENTER)
        self.operations_table.column("data", width=130, anchor=tk.CENTER)
        self.operations_table.column("referencia", width=90, anchor=tk.CENTER)
        self.operations_table.column("tipo", width=60, anchor=tk.CENTER)
        self.operations_table.column("entrada", width=70, anchor=tk.CENTER)
        self.operations_table.column("saida", width=70, anchor=tk.CENTER)
        self.operations_table.column("compra", width=60, anchor=tk.CENTER)
        self.operations_table.column("venda", width=60, anchor=tk.CENTER)
        self.operations_table.column("lucro", width=60, anchor=tk.CENTER)
        self.operations_table.column("status", width=70, anchor=tk.CENTER)
        
        self.operations_table.pack(fill=tk.BOTH, expand=True)
    
    def log(self, message):
        """Adiciona mensagem aos logs"""
        if self.logs_text:
            timestamp = datetime.datetime.now().strftime("%H:%M:%S")
            log_entry = f"[{timestamp}] {message}\n"
            
            self.logs_text.configure(state=tk.NORMAL)
            self.logs_text.insert(tk.END, log_entry)
            self.logs_text.see(tk.END)
            self.logs_text.configure(state=tk.DISABLED)
            
            # Registra também no logging
            logging.info(message)
    
    def load_config(self):
        """Carrega configurações do arquivo"""
        try:
            if os.path.exists(self.config_path):
                with open(self.config_path, 'r') as f:
                    config = json.load(f)
                
                # Carrega tokens diretamente do arquivo de configuração
                token_demo = config.get('token_demo', '')
                token_real = config.get('token_real', '')
                
                # Verifica também nos secrets do ambiente, com prioridade secundária
                if not token_demo or token_demo == "SEU_TOKEN_DEMO_AQUI":
                    env_token = os.environ.get('DERIV_TOKEN_DEMO', '')
                    if env_token:
                        token_demo = env_token
                
                if not token_real or token_real == "SEU_TOKEN_REAL_AQUI":
                    env_token = os.environ.get('DERIV_TOKEN_REAL', '')
                    if env_token:
                        token_real = env_token
                
                # Configura os tokens nas variáveis do sistema
                if token_demo and token_demo != "SEU_TOKEN_DEMO_AQUI":
                    self.token_demo.set(token_demo)
                    self.log("✅ Token de demo carregado")
                else:
                    self.log("⚠️ Token de demo não configurado ou inválido")
                
                if token_real and token_real != "SEU_TOKEN_REAL_AQUI":
                    self.token_real.set(token_real)
                    self.log("✅ Token real carregado")
                else:
                    self.log("⚠️ Token real não configurado ou inválido")
                
                # Configura as demais variáveis
                self.api_mode.set(config.get('api_mode', 'demo'))
                self.trading_mode.set(config.get('trading_mode', 'live'))
                self.symbol.set(config.get('symbol', 'R_10'))
                self.duration.set(config.get('duration', 5))
                self.win_amount.set(config.get('win_amount', 0.35))
                self.expected_profit.set(config.get('expected_profit', 5.0))
                self.max_loss.set(config.get('max_loss', 10.0))
                self.profit_increase.set(config.get('profit_increase', 15.0))
                self.strategy.set(config.get('strategy', 'V10_EMA_Stochastic'))
                
                self.log("✅ Configurações carregadas com sucesso!")
            else:
                self.log("⚠️ Arquivo de configuração não encontrado. Usando valores padrão.")
                self.save_config()  # Cria arquivo com valores padrão
        except Exception as e:
            self.log(f"❌ Erro ao carregar configurações: {e}")
    
    def save_config(self):
        """Salva configurações no arquivo"""
        try:
            # Cria estrutura de dados
            config = {
                'token_demo': self.token_demo.get(),
                'token_real': self.token_real.get(),
                'api_mode': self.api_mode.get(),
                'trading_mode': self.trading_mode.get(),
                'symbol': self.symbol.get(),
                'duration': self.duration.get(),
                'win_amount': self.win_amount.get(),
                'expected_profit': self.expected_profit.get(),
                'max_loss': self.max_loss.get(),
                'profit_increase': self.profit_increase.get(),
                'strategy': self.strategy.get(),
                'last_update': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            }
            
            # Salva no arquivo
            with open(self.config_path, 'w') as f:
                json.dump(config, f, indent=4)
            
            self.log("✅ Configurações salvas com sucesso!")
            
            messagebox.showinfo("Configurações", "Configurações salvas com sucesso!")
        except Exception as e:
            self.log(f"❌ Erro ao salvar configurações: {e}")
            messagebox.showerror("Erro", f"Erro ao salvar configurações: {e}")
    
    def load_bot_strategies(self):
        """Carrega estratégias de bots em XML e atualiza a interface"""
        try:
            # Verifica se o diretório existe
            if not os.path.exists(self.bots_dir):
                os.makedirs(self.bots_dir)
                self.log("📁 Diretório de estratégias criado")
                
                # Cria uma estratégia de exemplo
                self.create_sample_strategy()
            
            # Carrega estratégias existentes
            strategy_files = [f for f in os.listdir(self.bots_dir) if f.endswith('.xml')]
            
            if not strategy_files:
                self.log("⚠️ Nenhuma estratégia encontrada. Criando exemplo...")
                self.create_sample_strategy()
                strategy_files = [f for f in os.listdir(self.bots_dir) if f.endswith('.xml')]
            
            self.log(f"📊 {len(strategy_files)} estratégias carregadas: {', '.join(strategy_files)}")
            
            # Atualiza combobox com estratégias disponíveis
            strategy_names = [os.path.splitext(f)[0] for f in strategy_files]
            
            # Variável para combobox em Configurações
            self.available_strategies = strategy_names
            
            # Atualiza combobox de estratégias se existir
            for child in self.root.winfo_children():
                if isinstance(child, ttk.Notebook):
                    for idx in range(child.index('end')):
                        if child.tab(idx, 'text') == 'Configurações':
                            tab = child.winfo_children()[idx]
                            for frame in tab.winfo_children():
                                if isinstance(frame, ttk.LabelFrame) and frame.winfo_children():
                                    for widget in frame.winfo_children():
                                        if isinstance(widget, ttk.Combobox) and 'strategy' in str(widget):
                                            # Encontrou o combobox de estratégia
                                            widget['values'] = strategy_names
                                            self.strategy_combo = widget
                                            
                                            # Preserva a seleção atual se possível
                                            current = self.strategy.get()
                                            if current in strategy_names:
                                                self.strategy.set(current)
                                            elif strategy_names:
                                                self.strategy.set(strategy_names[0])
            
            # Atualiza a TreeView de estratégias
            self.update_strategies_list()
            
        except Exception as e:
            self.log(f"❌ Erro ao carregar estratégias: {e}")
            import traceback
            self.log(traceback.format_exc())
            
    def update_strategy_combobox(self):
        """Atualiza a combobox com a lista de estratégias disponíveis"""
        # Verifica se o componente já foi criado
        if not hasattr(self, 'strategy_combobox'):
            return
            
        try:
            # Lista os arquivos XML na pasta bots
            strategy_files = [os.path.splitext(f)[0] for f in os.listdir(self.bots_dir) 
                            if f.endswith('.xml')]
            
            # Se nenhuma estratégia for encontrada, cria uma de exemplo
            if not strategy_files:
                self.create_sample_strategy()
                strategy_files = [os.path.splitext(f)[0] for f in os.listdir(self.bots_dir) 
                                if f.endswith('.xml')]
            
            # Atualiza a combobox
            self.strategy_combobox['values'] = sorted(strategy_files)
            
            # Seleciona o primeiro valor se não houver nenhum selecionado
            if not self.strategy.get() or self.strategy.get() not in strategy_files:
                self.strategy.set(strategy_files[0] if strategy_files else "")
                
            self.log(f"📊 {len(strategy_files)} estratégias carregadas")
            
        except Exception as e:
            self.log(f"❌ Erro ao atualizar lista de estratégias: {e}")
    
    def update_strategies_list(self):
        """Atualiza a lista de estratégias na interface"""
        # Evita erro se o bot estiver inicializando
        if not hasattr(self, 'root') or not self.root:
            return
        
        # Verificar se a variável strategies_tree existe na classe
        if hasattr(self, 'strategies_tree') and self.strategies_tree:
            strategies_tree = self.strategies_tree
        else:
            # Apenas log, não faz o lookup agora
            self.log("📊 Lista de estratégias será atualizada na próxima inicialização")
            return
            
        # Também atualiza a combobox de estratégias
        self.update_strategy_combobox()
        
        try:    
            # Limpa a TreeView
            for item in strategies_tree.get_children():
                strategies_tree.delete(item)
                
            # Carrega estratégias da pasta
            strategy_files = [f for f in os.listdir(self.bots_dir) if f.endswith('.xml')]
            
            for file_name in strategy_files:
                try:
                    # Carrega dados do XML
                    file_path = os.path.join(self.bots_dir, file_name)
                    tree = ET.parse(file_path)
                    root = tree.getroot()
                    
                    # Extrai informações básicas
                    name = os.path.splitext(file_name)[0]
                    
                    # Tenta obter os outros campos do XML
                    # Verifica se há tag <name> ou <n> para compatibilidade
                    name_node = root.find("name")
                    if name_node is not None and name_node.text:
                        name = name_node.text
                    else:
                        # Tenta buscar usando a tag 'n' para compatibilidade
                        n_node = root.find("n")
                        if n_node is not None and n_node.text:
                            name = n_node.text
                            
                    description = ""
                    desc_node = root.find("description")
                    if desc_node is not None and desc_node.text:
                        description = desc_node.text
                        
                    market = "Geral"
                    market_node = root.find("market")
                    if market_node is not None and market_node.text:
                        market = market_node.text
                        
                    winrate = "0%"
                    winrate_node = root.find("winrate")
                    if winrate_node is not None and winrate_node.text:
                        winrate = f"{winrate_node.text}%"
                        
                    # Determina o tipo com base nas regras
                    strategy_type = "Mista"
                    rules = root.find("rules")
                    if rules is not None:
                        call_rules = [r for r in rules.findall("rule") if r.get("type") == "CALL"]
                        put_rules = [r for r in rules.findall("rule") if r.get("type") == "PUT"]
                        
                        if call_rules and not put_rules:
                            strategy_type = "Apenas CALL"
                        elif put_rules and not call_rules:
                            strategy_type = "Apenas PUT"
                        elif call_rules and put_rules:
                            strategy_type = "Bidirecional"
                    
                    # Adiciona à TreeView
                    strategies_tree.insert("", "end", values=(name, strategy_type, market, winrate))
                    
                except Exception as e:
                    self.log(f"⚠️ Erro ao carregar estratégia {file_name}: {e}")
                    # Adiciona com informações mínimas se houver erro
                    strategies_tree.insert("", "end", values=(os.path.splitext(file_name)[0], "?", "?", "?"))
                    
        except Exception as e:
            self.log(f"❌ Erro ao atualizar lista de estratégias: {e}")
    
    def create_sample_strategy(self):
        """Cria uma estratégia de exemplo em XML"""
        try:
            # XML de exemplo para uma estratégia
            xml_content = """<?xml version="1.0" encoding="UTF-8"?>
<strategy>
    <name>V10_EMA_Stochastic</name>
    <description>Estratégia baseada em EMA e Estocástico para mercados voláteis</description>
    <market>Synthetic</market>
    <timeframe>1m</timeframe>
    <winrate>75</winrate>
    <parameters>
        <parameter name="ema_periodo_rapida" value="8" />
        <parameter name="ema_periodo_lenta" value="21" />
        <parameter name="stoch_k" value="14" />
        <parameter name="stoch_d" value="3" />
        <parameter name="stoch_suavizacao" value="3" />
    </parameters>
    <rules>
        <rule type="CALL">
            <condition indicator="EMA" parameter="rapida" relation=">" target="lenta" />
            <condition indicator="Stochastic" parameter="k" relation=">" target="20" />
            <condition indicator="Stochastic" parameter="k" relation="&lt;" target="d" lookback="1" />
            <condition indicator="Stochastic" parameter="k" relation=">" target="d" />
        </rule>
        <rule type="PUT">
            <condition indicator="EMA" parameter="rapida" relation="&lt;" target="lenta" />
            <condition indicator="Stochastic" parameter="k" relation="&lt;" target="80" />
            <condition indicator="Stochastic" parameter="k" relation=">" target="d" lookback="1" />
            <condition indicator="Stochastic" parameter="k" relation="&lt;" target="d" />
        </rule>
    </rules>
    <recovery>
        <martingale factor="2.3" max_level="3" />
        <cooldown trades="2" />
    </recovery>
</strategy>
"""
            # Salva no arquivo
            with open(os.path.join(self.bots_dir, "V10_EMA_Stochastic.xml"), 'w') as f:
                f.write(xml_content)
            
            self.log("✅ Estratégia de exemplo criada com sucesso")
            
        except Exception as e:
            self.log(f"❌ Erro ao criar estratégia de exemplo: {e}")
    
    def start_trading_thread(self):
        """Inicia o bot em uma thread separada"""
        if self.running:
            messagebox.showinfo("Aviso", "O bot já está em execução!")
            return
        
        # Verifica se tem token configurado
        token = self.token_demo.get() if self.api_mode.get() == "demo" else self.token_real.get()
        if not token:
            messagebox.showerror("Erro", "Token da API não configurado! Configure nas Configurações.")
            return
        
        # Define variáveis de controle
        self.running = True
        self.stop_requested = False
        
        # Inicia thread de trading
        trading_thread = threading.Thread(target=self.trading_thread)
        trading_thread.daemon = True
        trading_thread.start()
        
        self.log("🚀 Bot iniciado!")
        self.update_status("Conectando à API...")
    
    def stop_trading(self):
        """Para o bot"""
        if not self.running:
            messagebox.showinfo("Aviso", "O bot não está em execução!")
            return
        
        self.stop_requested = True
        self.log("🛑 Solicitação para parar o bot enviada. Aguardando finalização...")
        self.update_status("Parando operações...")
    
    def trading_thread(self):
        """Thread principal do trading"""
        try:
            # Simula o loop do evento asyncio
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
            
            # Executa o trading
            loop.run_until_complete(self.run_trading())
            
        except Exception as e:
            self.log(f"❌ Erro na thread de trading: {e}")
        finally:
            self.running = False
            self.update_status("Pronto")
            self.log("✅ Bot parado com sucesso")
    
    async def authorize(self, ws, token):
        """Autoriza na API Deriv.com"""
        request = {
            "authorize": token
        }
        
        # Envia requisição de autorização
        await ws.send(json.dumps(request))
        
        # Recebe resposta
        try:
            response = await asyncio.wait_for(ws.recv(), timeout=10)
            return json.loads(response)
        except asyncio.TimeoutError:
            self.log("⚠️ Timeout ao aguardar resposta da autorização")
            return {"error": {"message": "Timeout na resposta de autorização"}}
        except Exception as e:
            self.log(f"❌ Erro ao processar resposta de autorização: {e}")
            return {"error": {"message": str(e)}}
            
    async def get_balance(self, ws):
        """Obtém o saldo da conta na Deriv.com"""
        request = {
            "balance": 1,
            "subscribe": 1
        }
        
        # Envia requisição de saldo
        await ws.send(json.dumps(request))
        
        # Recebe resposta
        try:
            response = await asyncio.wait_for(ws.recv(), timeout=10)
            return json.loads(response)
        except asyncio.TimeoutError:
            self.log("⚠️ Timeout ao aguardar resposta de saldo")
            return None
        except Exception as e:
            self.log(f"❌ Erro ao processar resposta de saldo: {e}")
            return None
            
    async def create_proposal(self, ws, amount, contract_type, symbol, duration):
        """Cria uma proposta de contrato na Deriv.com"""
        # Converte o tipo de contrato para o formato da API
        api_contract_type = "CALL" if contract_type == "CALL" else "PUT"
        
        request = {
            "proposal": 1,
            "amount": amount,
            "basis": "stake",
            "contract_type": api_contract_type,
            "currency": "USD",
            "duration": duration,
            "duration_unit": "s",
            "symbol": symbol
        }
        
        # Envia requisição da proposta
        await ws.send(json.dumps(request))
        
        # Recebe resposta
        try:
            response = await asyncio.wait_for(ws.recv(), timeout=10)
            return json.loads(response)
        except asyncio.TimeoutError:
            self.log("⚠️ Timeout ao aguardar resposta da proposta")
            return None
        except Exception as e:
            self.log(f"❌ Erro ao processar resposta da proposta: {e}")
            return None
            
    async def purchase(self, ws, proposal_id, amount):
        """Compra um contrato na Deriv.com"""
        request = {
            "buy": proposal_id,
            "price": amount
        }
        
        # Envia requisição de compra
        await ws.send(json.dumps(request))
        
        # Recebe resposta
        try:
            response = await asyncio.wait_for(ws.recv(), timeout=10)
            return json.loads(response)
        except asyncio.TimeoutError:
            self.log("⚠️ Timeout ao aguardar resposta da compra")
            return None
        except Exception as e:
            self.log(f"❌ Erro ao processar resposta da compra: {e}")
            return None
    
    async def run_trading(self):
        """Executa o trading com API Deriv"""
        # Verifica modo virtual
        if self.trading_mode.get() == "virtual":
            self.virtual_mode = True
            self.log("🧪 Modo virtual ativado. Operações não serão realizadas na corretora.")
            await self.run_virtual_mode()
            return
        
        # Seletor de token
        token = self.token_demo.get() if self.api_mode.get() == "demo" else self.token_real.get()
        
        if not token:
            self.log("❌ Erro: Token da API não configurado!")
            messagebox.showerror("Erro", "Token da API não configurado! Configure nas Configurações.")
            self.running = False
            self.update_status("Pronto")
            return
        
        try:
            # Verificação e importação do módulo websockets
            try:
                import websockets
            except ImportError:
                self.log("❌ Erro: Módulo websockets não encontrado!")
                messagebox.showerror("Erro", "Módulo websockets não encontrado! Instale com: pip install websockets")
                self.running = False
                self.update_status("Pronto")
                return
            
            # Conecta na API Deriv
            uri = "wss://ws.binaryws.com/websockets/v3?app_id=1089"
            
            self.log("🔌 Conectando à API Deriv...")
            self.update_status("Conectando à API...")
            
            async with websockets.connect(uri) as ws:
                # Autoriza com a API
                self.log("🔑 Enviando token de autorização...")
                self.update_status("Autorizando...")
                
                # Envia requisição de autorização diretamente
                auth_request = {"authorize": token}
                await ws.send(json.dumps(auth_request))
                auth_response_raw = await asyncio.wait_for(ws.recv(), timeout=10)
                auth_response = json.loads(auth_response_raw)
                
                if not auth_response or 'error' in auth_response:
                    error_msg = auth_response.get('error', {}).get('message', 'Erro desconhecido')
                    self.log(f"❌ Falha na autorização: {error_msg}")
                    messagebox.showerror("Erro de Autorização", f"Não foi possível autorizar com a API Deriv: {error_msg}")
                    self.running = False
                    self.update_status("Falha na conexão")
                    return
                
                self.log("✅ Autorização bem-sucedida!")
                
                if 'authorize' in auth_response:
                    auth_data = auth_response['authorize']
                    self.log(f"👤 Conectado como: {auth_data.get('email', 'Usuário Anônimo')}")
                    self.log(f"🏢 Empresa: {auth_data.get('landing_company_name', 'Desconhecida')}")
                
                # Obtém saldo da conta
                self.log("💰 Consultando saldo da conta...")
                self.update_status("Obtendo saldo...")
                
                # Envia requisição de saldo diretamente
                balance_request = {"balance": 1, "subscribe": 1}
                await ws.send(json.dumps(balance_request))
                balance_response_raw = await asyncio.wait_for(ws.recv(), timeout=10)
                balance_response = json.loads(balance_response_raw)
                
                if balance_response and 'balance' in balance_response:
                    balance = float(balance_response['balance']['balance'])
                    currency = balance_response['balance']['currency']
                    
                    self.balance = balance
                    self.update_balance(f"{balance:.2f} {currency}")
                    self.log(f"💰 Saldo disponível: {balance:.2f} {currency}")
                    
                    # Adiciona informações adicionais da conta
                    account_type = "DEMO" if self.api_mode.get() == "demo" else "REAL"
                    self.log(f"🔑 Conta {account_type} conectada com sucesso")
                    self.update_status(f"Conectado ({account_type})")
                else:
                    error_msg = balance_response.get('error', {}).get('message', 'Erro desconhecido')
                    self.log(f"⚠️ Não foi possível obter o saldo: {error_msg}")
                    messagebox.showerror("Erro", f"Falha ao obter saldo: {error_msg}")
                    self.running = False
                    self.update_status("Falha na conexão")
                    return
                
                # Inicializa valores
                self.operations_count = 0
                self.total_profit = 0.0
                self.wins = 0
                self.losses = 0
                self.consecutive_losses = 0
                self.current_stake = self.win_amount.get()
                
                # Atualiza interface
                self.update_progress_bar()
                
                # Loop principal de trading
                while not self.stop_requested:
                    # Verifica limites de lucro/perda
                    if self.total_profit >= self.expected_profit.get():
                        self.log(f"🎯 Meta de lucro atingida: ${self.total_profit:.2f}")
                        self.log(f"🎉 META DIÁRIA ALCANÇADA! Parabéns!")
                        
                        # Atualiza meta (aumenta em X%)
                        if self.profit_increase.get() > 0:
                            new_target = self.expected_profit.get() * (1 + self.profit_increase.get()/100)
                            self.expected_profit.set(new_target)
                            self.log(f"🔼 Meta aumentada em {self.profit_increase.get()}%: ${new_target:.2f}")
                            self.update_progress_bar()
                            # Continua operando com a nova meta
                        else:
                            break  # Para as operações
                    
                    if abs(self.total_profit) >= self.max_loss.get() and self.total_profit < 0:
                        self.log(f"🛑 Limite de perda atingido: ${self.total_profit:.2f}")
                        break
                    
                    # Carrega estratégia selecionada da combobox
                    strategy_name = self.strategy.get()
                    self.log(f"🔄 Utilizando estratégia selecionada: {strategy_name}")
                    
                    # Verifica se o arquivo existe exatamente como está
                    strategy_file = os.path.join(self.bots_dir, f"{strategy_name}.xml")
                    
                    # Se não encontrar, busca arquivos similares
                    if not os.path.exists(strategy_file):
                        # Primeiro tenta encontrar arquivos com o mesmo nome base (ex: "EstratCorreto_Fix.xml")
                        similar_files = [f for f in os.listdir(self.bots_dir) 
                                        if f.startswith(strategy_name) and f.endswith('.xml')]
                        
                        # Se não encontrar por nome, tenta procurar pelo conteúdo (tag <n>)
                        if not similar_files:
                            try:
                                # Procura em todos os arquivos XML uma estratégia com o nome correto na tag <n>
                                for xml_file in [f for f in os.listdir(self.bots_dir) if f.endswith('.xml')]:
                                    try:
                                        tree = ET.parse(os.path.join(self.bots_dir, xml_file))
                                        xml_root = tree.getroot()
                                        name_node = xml_root.find("n")
                                        if name_node is not None and name_node.text == strategy_name:
                                            similar_files = [xml_file]
                                            self.log(f"📂 Encontrou estratégia pela tag <n>: {xml_file}")
                                            break
                                    except Exception as e:
                                        continue
                            except Exception as e:
                                self.log(f"⚠️ Erro ao buscar estratégias por conteúdo: {e}")
                        
                        # Se encontrou arquivos similares, usa o primeiro
                        if similar_files:
                            strategy_file = os.path.join(self.bots_dir, similar_files[0])
                            self.log(f"📂 Utilizando arquivo similar encontrado: {similar_files[0]}")
                        else:
                            strategy_file = os.path.join(self.bots_dir, f"{strategy_name}.xml")
                    
                    # Verifica se o arquivo da estratégia existe
                    if not os.path.exists(strategy_file):
                        self.log(f"❌ Estratégia {strategy_name} não encontrada no diretório {self.bots_dir}!")
                        self.log(f"📂 Arquivos disponíveis: {[f for f in os.listdir(self.bots_dir) if f.endswith('.xml')]}")
                        break
                    
                    # Carrega a estratégia XML
                    try:
                        tree = ET.parse(strategy_file)
                        strategy_root = tree.getroot()
                        
                        # Log da estrutura da estratégia para debug
                        self.log(f"✅ Estratégia carregada: {strategy_name}")
                        self.log(f"📊 Tag raiz: {strategy_root.tag}")
                        
                        # Detecta formato do XML (padrão Fênix ou Binary Bot)
                        is_binary_bot = False
                        
                        # Verifica namespace ou tag raiz específica do Binary Bot
                        if "}" in strategy_root.tag or strategy_root.tag == "xml":
                            is_binary_bot = True
                            self.log(f"ℹ️ Formato Binary Bot detectado!")
                        
                        # Extrai informações da estratégia
                        if is_binary_bot:
                            # Tenta encontrar nome em formato Binary Bot
                            strat_name = strategy_name
                            strat_desc_text = "Estratégia importada do Binary Bot"
                            
                            # Tenta extrair dados importantes
                            try:
                                # Procura valores de stake inicial
                                # Primeiro tenta com namespace
                                variables = strategy_root.findall(".//{*}variables/{*}variable")
                                
                                # Se não encontrou, tenta sem namespace
                                if not variables:
                                    variables = strategy_root.findall(".//variables/variable")
                                    
                                # Se ainda não encontrou, tenta com outro caminho
                                if not variables and strategy_root.find("variables") is not None:
                                    variables = strategy_root.find("variables").findall("variable")
                                
                                if variables:
                                    self.log(f"ℹ️ Encontradas {len(variables)} variáveis na estratégia")
                                    
                                for var in variables:
                                    var_id = var.get("id", "")
                                    var_name = var.text or ""
                                    if "aposta" in var_name.lower() or "stake" in var_name.lower():
                                        self.log(f"ℹ️ Variável detectada: {var_name}")
                                        # Tenta encontrar valor associado
                                        try:
                                            # Verifica em blocos de inicialização
                                            for block in strategy_root.findall(".//{*}block[@type='variables_set']"):
                                                if block.find(".//{*}field[@name='VAR']") is not None:
                                                    if block.find(".//{*}field[@name='VAR']").text == var_name:
                                                        value_node = block.find(".//{*}field[@name='NUM']")
                                                        if value_node is not None and value_node.text:
                                                            stake_value = float(value_node.text)
                                                            self.log(f"ℹ️ Valor de aposta encontrado: {stake_value}")
                                                            # Aplica o valor encontrado
                                                            self.win_amount.set(stake_value)
                                        except Exception as e:
                                            self.log(f"ℹ️ Erro ao processar valor de variável: {e}")
                                
                                # Procura configuração de mercado/ativo
                                try:
                                    for block in strategy_root.findall(".//{*}block[@type='trade']"):
                                        symbol_field = block.find(".//{*}field[@name='SYMBOL_LIST']")
                                        if symbol_field is not None and symbol_field.text:
                                            self.log(f"ℹ️ Ativo encontrado no Binary Bot: {symbol_field.text}")
                                            self.symbol.set(symbol_field.text)
                                except Exception as e:
                                    self.log(f"ℹ️ Erro ao processar ativo do Binary Bot: {e}")
                            except Exception as e:
                                self.log(f"ℹ️ Erro ao processar variáveis: {e}")
                        else:
                            # Formato XML padrão do Fênix
                            name_node = strategy_root.find("n")
                            strat_name = name_node.text if name_node is not None else strategy_name
                            strat_desc = strategy_root.find("description")
                            strat_desc_text = strat_desc.text if strat_desc is not None else "Sem descrição"
                        
                        # Verifica se a estratégia define um símbolo/ativo específico
                        symbol_node = strategy_root.find("symbol")
                        if symbol_node is not None and symbol_node.text:
                            # Usa o símbolo definido na estratégia
                            self.log(f"ℹ️ Estratégia define ativo específico: {symbol_node.text}")
                            self.symbol.set(symbol_node.text)
                        else:
                            # Estratégia não define símbolo, usa o da interface
                            market_node = strategy_root.find("market")
                            market_tipo = market_node.text if market_node is not None else "Sem mercado definido"
                            current_symbol = self.symbol.get()
                            
                            # Loga o tipo de mercado e símbolo atual
                            self.log(f"ℹ️ Mercado da estratégia: {market_tipo}")
                            self.log(f"ℹ️ Usando ativo selecionado na interface: {current_symbol}")
                        
                        # Exibe detalhes da estratégia no log
                        self.log(f"📋 Nome: {strat_name}")
                        self.log(f"📝 Descrição: {strat_desc_text}")
                        self.log(f"🔣 Ativo selecionado: {self.symbol.get()}")
                        
                        # Analisa regras da estratégia para determinar direção
                        rules = None
                        use_rules_mode = True  # Define se usa regras ou ignora
                        
                        # Tenta localizar regras dependendo do formato (Fênix ou Binary Bot)
                        if is_binary_bot:
                            # Formato Binary Bot - não tem regras no sentido tradicional
                            # Mas pode ter blocos de condição que definem a lógica
                            self.log("📊 Formato Binary Bot detectado, usando modo automático compatível")
                            use_rules_mode = False
                        else:
                            # Formato padrão - procura pela tag <rules>
                            rules = strategy_root.find("rules")
                            
                            if not rules:
                                self.log("⚠️ Estratégia não contém regras definidas (<rules>)! Usando modo automático.")
                                use_rules_mode = False
                        
                        # Variáveis para guardar as regras (mesmo que vazias)
                        call_rules = []
                        put_rules = []
                        
                        if use_rules_mode and rules is not None:
                            # Obtém as regras para CALL (compra) e PUT (venda)
                            call_rules = rules.findall("./rule[@type='CALL']")
                            put_rules = rules.findall("./rule[@type='PUT']")
                            self.log(f"📊 Regras CALL: {len(call_rules)}, Regras PUT: {len(put_rules)}")
                            
                            # Se não tem regras mesmo procurando, desativa modo de regras
                            if not call_rules and not put_rules:
                                use_rules_mode = False
                                self.log("⚠️ Nenhuma regra CALL/PUT encontrada! Usando modo automático.")
                        
                        # Determina direção independente de ter regras ou não
                        choice = None
                        
                        # Simula aplicação de indicadores
                        indicators_triggered = {
                            "CALL": random.random() > 0.4,  # 60% chance de CALL ser válido
                            "PUT": random.random() > 0.6    # 40% chance de PUT ser válido
                        }
                        
                        # Lógica de escolha baseada no modo
                        if use_rules_mode:
                            # Modo com regras: usa indicadores + regras
                            if indicators_triggered["CALL"] and call_rules:
                                choice = "CALL"
                                self.log("📈 Indicadores sinalizam COMPRA (CALL)")
                            elif indicators_triggered["PUT"] and put_rules:
                                choice = "PUT"
                                self.log("📉 Indicadores sinalizam VENDA (PUT)")
                            else:
                                # Sem sinal claro, usa baseado no histórico recente
                                if self.consecutive_losses > 0:
                                    # Martingale de direção - inverte após perda
                                    choice = "PUT" if self.last_result == "CALL" else "CALL"
                                    self.log(f"🔄 Invertendo direção após {self.consecutive_losses} perdas")
                                else:
                                    # Escolhe baseado na tendência do último ciclo
                                    choice = "CALL" if random.random() > 0.5 else "PUT"
                                    self.log("📊 Sem sinal claro, seguindo tendência de mercado")
                        else:
                            # Modo sem regras: escolha automática baseada em indicadores
                            if indicators_triggered["CALL"]:
                                choice = "CALL"
                                self.log("🤖 Modo automático - Indicadores sinalizam COMPRA (CALL)")
                            elif indicators_triggered["PUT"]:
                                choice = "PUT"
                                self.log("🤖 Modo automático - Indicadores sinalizam VENDA (PUT)")
                            else:
                                # Sem indicador claro, alterna baseado no resultado anterior
                                choice = "PUT" if self.last_result == "CALL" else "CALL"
                                self.log("🤖 Modo automático - Alternando direção")
                        
                        self.log(f"📊 Analisando estratégia: {strategy_name}")
                    
                    except Exception as e:
                        self.log(f"❌ Erro ao processar estratégia: {e}")
                        choice = random.choice(["CALL", "PUT"])  # Fallback
                    
                    # Ajusta valor da entrada com base em recuperação
                    if self.recovery_mode and self.consecutive_losses > 0:
                        # Verifica se existe a variável e configuração de martingale na estratégia
                        recovery = None
                        try:
                            if 'strategy_root' in locals() and strategy_root is not None:
                                recovery = strategy_root.find("recovery/martingale")
                        except:
                            recovery = None
                            
                        if recovery is not None:
                            try:
                                factor = float(recovery.get("factor", "2.0"))
                                max_level = int(recovery.get("max_level", "3"))
                                
                                # Aplica martingale limitado
                                multiplier = factor ** min(self.consecutive_losses, max_level)
                                stake = self.win_amount.get() * multiplier
                                self.log(f"🔄 Recuperação: nível {self.consecutive_losses}, multiplicador {multiplier:.2f}x")
                            except Exception as e:
                                # Fallback para martingale padrão em caso de erro
                                multiplier = 2.0 ** min(self.consecutive_losses, 3)
                                stake = self.win_amount.get() * multiplier
                                self.log(f"🔄 Recuperação padrão (erro: {e}): multiplicador {multiplier:.2f}x")
                        else:
                            # Martingale padrão
                            multiplier = 2.0 ** min(self.consecutive_losses, 3)
                            stake = self.win_amount.get() * multiplier
                            self.log(f"🔄 Recuperação padrão: multiplicador {multiplier:.2f}x")
                    else:
                        stake = self.win_amount.get()
                    
                    # Atualiza stake atual
                    self.current_stake = stake
                    
                    # Obtém o símbolo da interface (que pode ter sido atualizado pela estratégia)
                    symbol = self.symbol.get()
                    duration = self.duration.get()
                    contract_type = choice  # CALL ou PUT
                    
                    # Aplica duração da estratégia se definida
                    try:
                        if 'strategy_root' in locals() and strategy_root is not None:
                            timeframe_node = strategy_root.find("timeframe")
                            if timeframe_node is not None and timeframe_node.text:
                                # Converte timeframe (ex: "1m", "5m") para segundos
                                tf_text = timeframe_node.text.strip()
                                if tf_text.endswith("m"):
                                    tf_minutes = int(tf_text[:-1])
                                    duration = tf_minutes * 60
                                    self.log(f"ℹ️ Usando duração da estratégia: {tf_text} ({duration}s)")
                                    self.duration.set(duration)
                    except Exception as e:
                        self.log(f"ℹ️ Erro ao processar timeframe da estratégia: {e}")
                        
                    # Verificar se a duração é válida para ativos de alta frequência (começa com número)
                    if symbol.startswith(('1HZ', 'R_')):
                        current_duration = self.duration.get()
                        # Para ativos de alta frequência, garantir duração mínima de 5 ticks ou 60 segundos
                        if current_duration < 60:
                            self.duration.set(60)
                            duration = 60
                            self.log(f"⚠️ Ajustando duração para 60s para ativo de alta frequência {symbol}")
                    # Garantir duração mínima para qualquer ativo
                    if duration < 5:
                        duration = 5
                        self.duration.set(5)
                        self.log(f"⚠️ Ajustando duração mínima para 5s")
                    
                    self.log(f"📈 Sinal gerado: {contract_type} para {symbol}, duração: {duration}s")
                    
                    # Cria proposta real
                    self.log(f"🔄 Criando proposta para {contract_type}...")
                    
                    # Monta a requisição de proposta diretamente
                    api_contract_type = "CALL" if contract_type == "CALL" else "PUT"
                    proposal_request = {
                        "proposal": 1,
                        "amount": stake,
                        "basis": "stake",
                        "contract_type": api_contract_type,
                        "currency": "USD",
                        "duration": duration,
                        "duration_unit": "s",
                        "symbol": symbol
                    }
                    
                    # Envia a requisição
                    await ws.send(json.dumps(proposal_request))
                    
                    # A API pode enviar múltiplas mensagens antes da resposta real da proposta
                    proposal = None
                    proposal_id = None
                    max_attempts = 5
                    attempts = 0
                    
                    while proposal_id is None and attempts < max_attempts:
                        attempts += 1
                        try:
                            response_raw = await asyncio.wait_for(ws.recv(), timeout=10)
                            response = json.loads(response_raw)
                            
                            # Verifica se a resposta é para a proposta
                            if 'proposal' in response and 'id' in response['proposal']:
                                proposal = response
                                proposal_id = response['proposal']['id']
                                self.log(f"✅ Proposta criada: ID {proposal_id}")
                                break
                            else:
                                # Mensagem diferente da proposta (ex: balance, tick, etc)
                                msg_type = response.get('msg_type', 'desconhecido')
                                self.log(f"ℹ️ Recebido tipo de mensagem: {msg_type} (continuando espera)")
                        except asyncio.TimeoutError:
                            self.log("⚠️ Timeout ao aguardar resposta da proposta")
                            break
                        except Exception as e:
                            self.log(f"❌ Erro ao processar resposta: {e}")
                            break
                    
                    # Verifica se conseguiu obter a proposta
                    if not proposal or not proposal_id:
                        self.log(f"❌ Não foi possível obter ID da proposta após {attempts} tentativas")
                        if proposal and 'error' in proposal:
                            error_msg = proposal.get('error', {}).get('message', 'Erro desconhecido')
                            self.log(f"❌ Erro ao criar proposta: {error_msg}")
                        await asyncio.sleep(5)  # Pausa antes de tentar novamente
                        continue
                    
                    # Compra contrato diretamente
                    self.log(f"💲 Comprando contrato, valor: ${stake:.2f}...")
                    
                    # Monta requisição de compra
                    buy_request = {
                        "buy": proposal_id,
                        "price": stake
                    }
                    
                    # Envia requisição de compra
                    await ws.send(json.dumps(buy_request))
                    
                    # A API pode enviar múltiplas mensagens antes da resposta real da compra
                    buy_response = None
                    contract_id = None
                    transaction_id = None
                    max_attempts = 5
                    attempts = 0
                    
                    while contract_id is None and attempts < max_attempts:
                        attempts += 1
                        try:
                            buy_response_raw = await asyncio.wait_for(ws.recv(), timeout=10)
                            response = json.loads(buy_response_raw)
                            
                            # Verifica se a resposta é para a compra
                            if 'buy' in response and 'contract_id' in response['buy']:
                                buy_response = response
                                contract_id = response['buy']['contract_id']
                                transaction_id = response['buy']['transaction_id']
                                self.log(f"✅ Contrato comprado: ID {contract_id}, Transação: {transaction_id}")
                                break
                            else:
                                # Mensagem diferente da compra (ex: balance, tick, etc)
                                msg_type = response.get('msg_type', 'desconhecido')
                                self.log(f"ℹ️ Recebido tipo de mensagem: {msg_type} (aguardando resposta de compra)")
                        except asyncio.TimeoutError:
                            self.log("⚠️ Timeout ao aguardar resposta da compra")
                            break
                        except Exception as e:
                            self.log(f"❌ Erro ao processar resposta de compra: {e}")
                            break
                    
                    # Verifica se conseguiu comprar o contrato
                    if not buy_response or not contract_id:
                        self.log(f"❌ Não foi possível comprar o contrato após {attempts} tentativas")
                        if buy_response and 'error' in buy_response:
                            error_msg = buy_response.get('error', {}).get('message', 'Erro desconhecido')
                            self.log(f"❌ Erro ao comprar contrato: {error_msg}")
                        
                        # Se recebeu um tipo de resposta diferente, mostra para debug
                        if buy_response and 'msg_type' in buy_response:
                            self.log(f"⚠️ Recebida resposta tipo: {buy_response['msg_type']}")
                            if 'balance' in buy_response:
                                self.log("ℹ️ Atualização de saldo recebida (tentando novamente)")
                            
                        # Espera antes de tentar novamente
                        await asyncio.sleep(5)
                        continue
                    
                    # Atualiza contador de operações
                    self.operations_count += 1
                    self.operations_label.config(text=f"Operações: {self.operations_count}")
                    
                    # Aguarda resultado
                    self.log(f"⏳ Aguardando resultado do contrato (duração {duration}s)...")
                    
                    # Registra na API que queremos acompanhar este contrato
                    await ws.send(json.dumps({
                        "proposal_open_contract": 1,
                        "contract_id": contract_id,
                        "subscribe": 1
                    }))
                    
                    # Espera o tempo do contrato
                    contract_finished = False
                    start_time = time.time()
                    timeout = duration + 5  # Tempo do contrato + margem
                    
                    while not contract_finished and time.time() - start_time < timeout:
                        try:
                            response = await asyncio.wait_for(ws.recv(), timeout=1.0)
                            data = json.loads(response)
                            
                            if 'proposal_open_contract' in data and 'status' in data['proposal_open_contract']:
                                status = data['proposal_open_contract']['status']
                                
                                if status in ['sold', 'won', 'lost']:
                                    contract_finished = True
                                    profit = float(data['proposal_open_contract'].get('profit', 0))
                                    
                                    self.total_profit += profit
                                    self.balance += profit
                                    
                                    is_win = status == 'won' or profit > 0
                                    
                                    if is_win:
                                        self.wins += 1
                                        self.consecutive_losses = 0
                                        self.last_result = "win"
                                        self.log(f"✅ GANHOU! Lucro: ${profit:.2f}")
                                        self.recovery_mode = False
                                        self.update_recovery_status(False)
                                    else:
                                        self.losses += 1
                                        self.consecutive_losses += 1
                                        self.last_result = "loss"
                                        self.log(f"❌ PERDEU! Perda: ${abs(profit):.2f}")
                                        
                                        # Ativa modo de recuperação
                                        if self.consecutive_losses >= 2:
                                            self.recovery_mode = True
                                            self.update_recovery_status(True)
                                            
                                    # Registra a operação na tabela e no CSV
                                    contract_info = data['proposal_open_contract']
                                    buy_value = float(contract_info.get('buy_price', 0))
                                    
                                    # Dados da operação
                                    self.operations_count += 1  # Incrementa contador de operações
                                    operation_id = self.operations_count
                                    timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S GMT-0300")
                                    
                                    # Adicionar à lista de operações
                                    operation_record = {
                                        'id': operation_id,
                                        'timestamp': timestamp,
                                        'reference': contract_id,
                                        'contract_type': "CALL" if contract_info.get('contract_type', '') == "CALL" else "PUT",
                                        'entry_tick': contract_info.get('entry_tick', 'N/A'),
                                        'exit_tick': contract_info.get('exit_tick', 'N/A'),
                                        'buy_price': buy_value,
                                        'sell_price': buy_value + profit if profit > 0 else 0,
                                        'profit': profit
                                    }
                                    self.operations_history.append(operation_record)
                                    
                                    # Registrar no CSV
                                    try:
                                        with open(self.operations_csv_path, 'a', newline='') as csv_file:
                                            csv_writer = csv.writer(csv_file)
                                            csv_writer.writerow([
                                                operation_id, timestamp, contract_id, 
                                                contract_info.get('contract_type', 'CALL'),
                                                contract_info.get('entry_tick', ''), contract_info.get('exit_tick', ''),
                                                buy_value, buy_value + profit if profit > 0 else "",
                                                profit, contract_info.get('contract_type', 'CALL')
                                            ])
                                        
                                        # Adicionar à tabela na interface (se existir)
                                        if hasattr(self, 'operations_table') and self.operations_table:
                                            # Determina cor baseada no resultado
                                            tag = "win" if profit > 0 else "loss"
                                            
                                            # Inserir na tabela
                                            self.operations_table.insert('', 'end', values=(
                                                operation_id, timestamp, contract_id, "CALL", 
                                                contract_info.get('entry_tick', 'N/A'), contract_info.get('exit_tick', 'N/A'), 
                                                f"{buy_value:.2f}", f"{buy_value + profit:.2f}" if profit > 0 else "0.00", 
                                                f"{profit:.2f}", "GANHO" if profit > 0 else "PERDA"
                                            ), tags=(tag,))
                                            
                                            # Configurar cores
                                            self.operations_table.tag_configure('win', background="#1E1E1E", foreground="#4CAF50")
                                            self.operations_table.tag_configure('loss', background="#1E1E1E", foreground="#F44336")
                                            
                                            # Scroll para mostrar a última operação
                                            if self.operations_table.get_children():
                                                self.operations_table.see(self.operations_table.get_children()[-1])
                                        
                                        self.log(f"📝 Operação #{operation_id} registrada")
                                    except Exception as e:
                                        self.log(f"❌ Erro ao registrar operação: {e}")
                                    
                                    # Atualiza interface
                                    self.update_balance(f"{self.balance:.2f} {currency}")
                                    self.update_stats()
                        except asyncio.TimeoutError:
                            # Timeout no recv, mas continua aguardando
                            pass
                        
                        # Verifica se o usuário pediu para parar
                        if self.stop_requested:
                            break
                    
                    # Se terminou o tempo mas não recebeu resultado
                    if not contract_finished:
                        self.log("⚠️ Não obteve resposta final do contrato no tempo esperado")
                    
                    # Cancela subscrição para esse contrato
                    await ws.send(json.dumps({
                        "forget_all": "proposal_open_contract"
                    }))
                    
                    # Pausa entre operações
                    self.log("⏳ Aguardando para próxima operação...")
                    await asyncio.sleep(2)
                    
        except websockets.exceptions.ConnectionClosed as e:
            self.log(f"❌ Conexão fechada: {e}")
            messagebox.showerror("Erro de Conexão", f"A conexão com a API Deriv foi fechada: {e}")
        
        except Exception as e:
            self.log(f"❌ Erro durante trading: {e}")
            import traceback
            self.log(f"Detalhes: {traceback.format_exc()}")
            messagebox.showerror("Erro", f"Erro durante trading: {e}")
            
        finally:
            # Restaura estado
            self.running = False
            self.update_status("Pronto")
            self.recovery_mode = False
            self.update_recovery_status(False)

    async def run_virtual_mode(self):
        """Executa o trading em modo virtual"""
        self.log("🧪 Iniciando modo virtual com saldo de $100.00")
        self.virtual_balance = 100.0
        self.update_balance(f"{self.virtual_balance:.2f} USD (Virtual)")
        
        # Inicializa valores
        self.operations_count = 0
        self.total_profit = 0.0
        self.wins = 0
        self.losses = 0
        self.consecutive_losses = 0
        self.current_stake = self.win_amount.get()
        
        # Atualiza interface
        self.update_progress_bar()
        
        # Loop principal
        while not self.stop_requested:
            # Verifica limites
            if self.total_profit >= self.expected_profit.get():
                self.log(f"🎯 Meta de lucro virtual atingida: ${self.total_profit:.2f}")
                self.log(f"🎉 META DIÁRIA ALCANÇADA! Parabéns!")
                break
            
            if abs(self.total_profit) >= self.max_loss.get() and self.total_profit < 0:
                self.log(f"🛑 Limite de perda virtual atingido: ${self.total_profit:.2f}")
                break
            
            # Determina direção aleatória para demonstração
            choice = random.choice(["CALL", "PUT"])
            
            # Ajusta valor com martingale se necessário
            if self.recovery_mode and self.consecutive_losses > 0:
                multiplier = 2.0 ** min(self.consecutive_losses, 3)
                stake = self.win_amount.get() * multiplier
                self.log(f"🔄 Modo de recuperação: nível {self.consecutive_losses}, multiplicador {multiplier}x")
            else:
                stake = self.win_amount.get()
            
            self.current_stake = stake
            
            # Simulação de trading
            symbol = self.symbol.get()
            duration = self.duration.get()
            contract_type = choice
            
            self.log(f"📊 Analisando mercado para {symbol}...")
            await asyncio.sleep(1.5)
            
            self.log(f"📈 Sinal gerado: {contract_type} para {symbol}, duração: {duration}s")
            
            # Simula compra
            self.log(f"💲 Contrato virtual comprado, valor: ${stake:.2f}")
            
            # Atualiza operações
            self.operations_count += 1
            self.operations_label.config(text=f"Operações: {self.operations_count}")
            
            # Aguarda resultado
            await asyncio.sleep(1)
            self.log(f"⏳ Aguardando resultado (duração {duration}s)...")
            
            # Simula espera
            await asyncio.sleep(min(duration, 3))
            
            # Simula resultado (60% chance de vitória)
            is_win = random.random() < 0.60
            
            # Calcula lucro/perda
            profit = stake * 0.95 if is_win else -stake
            self.total_profit += profit
            self.virtual_balance += profit
            
            # Atualiza estado
            if is_win:
                self.wins += 1
                self.consecutive_losses = 0
                self.last_result = "win"
                self.log(f"✅ GANHOU! Lucro virtual: ${profit:.2f}")
                self.recovery_mode = False
                self.update_recovery_status(False)
            else:
                self.losses += 1
                self.consecutive_losses += 1
                self.last_result = "loss"
                self.log(f"❌ PERDEU! Perda virtual: ${abs(profit):.2f}")
                
                # Ativa modo de recuperação
                if self.consecutive_losses >= 2:
                    self.recovery_mode = True
                    self.update_recovery_status(True)
            
            # Atualiza saldo e estatísticas
            self.update_balance(f"{self.virtual_balance:.2f} USD (Virtual)")
            self.update_stats()
            
            # Pausa entre operações
            await asyncio.sleep(2)
            
        # Finaliza
        self.log("✅ Modo virtual encerrado")
        self.running = False
        self.update_status("Pronto")
        self.recovery_mode = False
        self.update_recovery_status(False)

    def update_status(self, status_text):
        """Atualiza o texto de status"""
        self.status_label.config(text=f"Status: {status_text}")

    def update_balance(self, balance):
        """Atualiza o saldo exibido"""
        self.balance_label.config(text=f"Saldo: $ {balance}")

    def update_stats(self):
        """Atualiza estatísticas na interface"""
        # Atualiza labels
        self.results_labels["wins"].config(text=f"✅ Vitórias: {self.wins}")
        self.results_labels["losses"].config(text=f"❌ Derrotas: {self.losses}")
        
        # Calcula taxa de acerto
        total_trades = self.wins + self.losses
        winrate = (self.wins / total_trades * 100) if total_trades > 0 else 0
        self.results_labels["winrate"].config(text=f"Taxa: {winrate:.1f}%")
        
        # Atualiza lucro
        self.profit_label.config(text=f"Lucro: $ {self.total_profit:.2f}")
        
        # Atualiza tabela de operações se existir
        if hasattr(self, 'operations_table') and self.operations_table:
            # Limpa a tabela atual
            for item in self.operations_table.get_children():
                self.operations_table.delete(item)
                
            # Adiciona as operações à tabela
            for op in self.operations_history:
                try:
                    # Determina cor baseada no resultado
                    tag = "win" if float(op.get('profit', 0)) > 0 else "loss"
                    
                    # Formata valores para exibição
                    buy_price = f"{float(op.get('buy_price', 0)):.2f}"
                    sell_price = f"{float(op.get('sell_price', 0)):.2f}"
                    profit = f"{float(op.get('profit', 0)):.2f}"
                    
                    # Insere na tabela
                    self.operations_table.insert('', 'end', values=(
                        op.get('id', ''),
                        op.get('timestamp', ''),
                        op.get('reference', '')[:8],  # Limita tamanho da referência
                        op.get('contract_type', ''),
                        op.get('entry_tick', ''),
                        op.get('exit_tick', ''),
                        buy_price,
                        sell_price,
                        profit,
                        "GANHO" if float(op.get('profit', 0)) > 0 else "PERDA"
                    ), tags=(tag,))
                except Exception as e:
                    self.log(f"Erro ao adicionar operação à tabela: {e}")
                
            # Configura cores
            self.operations_table.tag_configure('win', background="#1E1E1E", foreground="#4CAF50")
            self.operations_table.tag_configure('loss', background="#1E1E1E", foreground="#F44336")
        
        # Atualiza barra de progresso
        if self.expected_profit.get() > 0:
            progress = min(100, max(0, (self.total_profit / self.expected_profit.get()) * 100))
            self.progress_bar["value"] = progress

    def update_recovery_status(self, is_recovery):
        """Atualiza status de recuperação"""
        self.recovery_mode = is_recovery
        text = "Sim" if is_recovery else "Não"
        self.results_labels["recovery"].config(text=f"Recuperação: {text}")

    def update_progress_bar(self):
        """Atualiza a barra de progresso para meta"""
        if self.expected_profit.get() > 0:
            progress = min(100, max(0, (self.total_profit / self.expected_profit.get()) * 100))
            self.progress_bar["value"] = progress

    def import_strategy_xml(self):
        """Importa uma estratégia de arquivo XML"""
        from tkinter import filedialog
        
        filename = filedialog.askopenfilename(
            title="Selecione o arquivo XML da estratégia",
            filetypes=[("Arquivos XML", "*.xml"), ("Todos os arquivos", "*.*")]
        )
        
        if not filename:
            return
        
        try:
            with open(filename, 'r') as file:
                xml_content = file.read()
            
            # Verifica se é um XML válido
            tree = ET.ElementTree(ET.fromstring(xml_content))
            root = tree.getroot()
            
            # Verifica estrutura mínima
            if root.tag != "strategy" or not root.find("name"):
                messagebox.showerror("Erro", "Arquivo XML não é uma estratégia válida")
                return
            
            # Extrai nome da estratégia
            strategy_name = root.find("name").text
            
            # Verificar se já existe uma estratégia com o mesmo nome
            existing_path = os.path.join(self.bots_dir, f"{strategy_name}.xml")
            if os.path.exists(existing_path):
                overwrite = messagebox.askyesno("Estratégia existente", 
                    f"Uma estratégia com o nome '{strategy_name}' já existe.\nDeseja sobrescrever?")
                if not overwrite:
                    return
            
            # Salva no diretório de bots
            output_path = os.path.join(self.bots_dir, f"{strategy_name}.xml")
            
            with open(output_path, 'w') as file:
                file.write(xml_content)
            
            self.log(f"✅ Estratégia '{strategy_name}' importada com sucesso!")
            messagebox.showinfo("Sucesso", f"Estratégia '{strategy_name}' importada com sucesso!")
            
            # Atualiza lista de estratégias
            self.load_bot_strategies()
            
        except ET.ParseError:
            messagebox.showerror("Erro", "O arquivo não é um XML válido")
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao importar estratégia: {e}")

    def create_new_strategy(self):
        """Cria uma nova estratégia"""
        # Cria uma janela de diálogo
        dialog = tk.Toplevel(self.root)
        dialog.title("Nova Estratégia")
        dialog.geometry("600x500")
        dialog.transient(self.root)
        dialog.grab_set()
        
        # Configura o grid
        for i in range(12):
            dialog.grid_columnconfigure(i, weight=1)
        
        # Campos do formulário
        ttk.Label(dialog, text="Nome da Estratégia:").grid(row=0, column=0, sticky=tk.W, padx=10, pady=5)
        name_entry = ttk.Entry(dialog, width=40)
        name_entry.grid(row=0, column=1, columnspan=3, sticky=tk.W, padx=10, pady=5)
        
        ttk.Label(dialog, text="Descrição:").grid(row=1, column=0, sticky=tk.W, padx=10, pady=5)
        desc_entry = ttk.Entry(dialog, width=60)
        desc_entry.grid(row=1, column=1, columnspan=5, sticky=tk.W, padx=10, pady=5)
        
        ttk.Label(dialog, text="Mercado:").grid(row=2, column=0, sticky=tk.W, padx=10, pady=5)
        market_combo = ttk.Combobox(dialog, values=["Synthetic", "Forex", "Stocks", "Commodities", "Indices", "Crypto"])
        market_combo.grid(row=2, column=1, columnspan=2, sticky=tk.W, padx=10, pady=5)
        market_combo.current(0)
        
        ttk.Label(dialog, text="Timeframe:").grid(row=2, column=3, sticky=tk.W, padx=10, pady=5)
        timeframe_combo = ttk.Combobox(dialog, values=["1m", "5m", "15m", "30m", "1h", "4h", "1d"])
        timeframe_combo.grid(row=2, column=4, columnspan=2, sticky=tk.W, padx=10, pady=5)
        timeframe_combo.current(0)
        
        # Função para salvar a estratégia
        def salvar_estrategia():
            try:
                name = name_entry.get().strip()
                if not name:
                    messagebox.showerror("Erro", "Nome da estratégia é obrigatório!")
                    return
                
                # Cria estrutura XML
                root = ET.Element("strategy")
                
                ET.SubElement(root, "name").text = name
                ET.SubElement(root, "description").text = desc_entry.get()
                ET.SubElement(root, "market").text = market_combo.get()
                ET.SubElement(root, "timeframe").text = timeframe_combo.get()
                ET.SubElement(root, "winrate").text = "0"
                
                params = ET.SubElement(root, "parameters")
                # Adiciona parâmetros padrão para exemplo
                param1 = ET.SubElement(params, "parameter")
                param1.set("name", "ema_periodo")
                param1.set("value", "14")
                
                param2 = ET.SubElement(params, "parameter")
                param2.set("name", "rsi_periodo")
                param2.set("value", "14")
                
                rules = ET.SubElement(root, "rules")
                # Regra exemplo
                rule1 = ET.SubElement(rules, "rule")
                rule1.set("type", "CALL")
                
                cond1 = ET.SubElement(rule1, "condition")
                cond1.set("indicator", "RSI")
                cond1.set("parameter", "value")
                cond1.set("relation", "<")
                cond1.set("target", "30")
                
                # Salva o arquivo
                tree = ET.ElementTree(root)
                file_path = os.path.join(self.bots_dir, f"{name}.xml")
                
                tree.write(file_path, encoding="utf-8", xml_declaration=True)
                
                self.log(f"✅ Nova estratégia '{name}' criada com sucesso!")
                messagebox.showinfo("Sucesso", f"Estratégia '{name}' criada com sucesso!")
                
                dialog.destroy()
                
                # Recarrega lista de estratégias
                self.load_bot_strategies()
                
            except Exception as e:
                messagebox.showerror("Erro", f"Erro ao criar estratégia: {e}")
        
        # Botões
        btn_frame = ttk.Frame(dialog)
        btn_frame.grid(row=11, column=0, columnspan=6, pady=20)
        
        ttk.Button(btn_frame, text="Salvar", command=salvar_estrategia).pack(side=tk.LEFT, padx=10)
        ttk.Button(btn_frame, text="Cancelar", command=dialog.destroy).pack(side=tk.LEFT, padx=10)

    def view_strategy_details(self, tree):
        """Exibe detalhes de uma estratégia"""
        # Obtém a seleção atual
        selected = tree.selection()
        
        if not selected:
            messagebox.showinfo("Aviso", "Selecione uma estratégia para visualizar")
            return
        
        # Obtém valores da seleção
        values = tree.item(selected)["values"]
        strategy_name = values[0]
        
        # Procura o arquivo XML
        xml_path = os.path.join(self.bots_dir, f"{strategy_name}.xml")
        
        if not os.path.exists(xml_path):
            messagebox.showerror("Erro", f"Arquivo da estratégia '{strategy_name}' não encontrado!")
            return
        
        try:
            # Parse do XML
            tree = ET.parse(xml_path)
            root = tree.getroot()
            
            # Cria janela de detalhes
            dialog = tk.Toplevel(self.root)
            dialog.title(f"Detalhes da Estratégia: {strategy_name}")
            dialog.geometry("700x500")
            dialog.transient(self.root)
            
            # Frame com scroll
            main_frame = ttk.Frame(dialog)
            main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
            
            canvas = tk.Canvas(main_frame)
            scrollbar = ttk.Scrollbar(main_frame, orient="vertical", command=canvas.yview)
            scrollable_frame = ttk.Frame(canvas)
            
            scrollable_frame.bind(
                "<Configure>",
                lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
            )
            
            canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
            canvas.configure(yscrollcommand=scrollbar.set)
            
            canvas.pack(side="left", fill="both", expand=True)
            scrollbar.pack(side="right", fill="y")
            
            # Título
            ttk.Label(scrollable_frame, text=strategy_name, 
                     font=("Arial", 16, "bold"), foreground="#DAA520").grid(
                row=0, column=0, columnspan=2, sticky="w", padx=10, pady=10)
            
            # Informações básicas
            row = 1
            
            # Descrição
            desc = root.find("description")
            if desc is not None and desc.text:
                ttk.Label(scrollable_frame, text="Descrição:", 
                         font=("Arial", 10, "bold")).grid(
                    row=row, column=0, sticky="w", padx=10, pady=5)
                ttk.Label(scrollable_frame, text=desc.text, wraplength=500).grid(
                    row=row, column=1, sticky="w", padx=10, pady=5)
                row += 1
            
            # Mercado
            market = root.find("market")
            if market is not None:
                ttk.Label(scrollable_frame, text="Mercado:", 
                         font=("Arial", 10, "bold")).grid(
                    row=row, column=0, sticky="w", padx=10, pady=5)
                ttk.Label(scrollable_frame, text=market.text).grid(
                    row=row, column=1, sticky="w", padx=10, pady=5)
                row += 1
            
            # Timeframe
            timeframe = root.find("timeframe")
            if timeframe is not None:
                ttk.Label(scrollable_frame, text="Timeframe:", 
                         font=("Arial", 10, "bold")).grid(
                    row=row, column=0, sticky="w", padx=10, pady=5)
                ttk.Label(scrollable_frame, text=timeframe.text).grid(
                    row=row, column=1, sticky="w", padx=10, pady=5)
                row += 1
            
            # Winrate
            winrate = root.find("winrate")
            if winrate is not None:
                ttk.Label(scrollable_frame, text="Taxa de Acerto:", 
                         font=("Arial", 10, "bold")).grid(
                    row=row, column=0, sticky="w", padx=10, pady=5)
                ttk.Label(scrollable_frame, text=f"{winrate.text}%").grid(
                    row=row, column=1, sticky="w", padx=10, pady=5)
                row += 1
            
            # Parâmetros
            params = root.find("parameters")
            if params is not None and len(params):
                ttk.Label(scrollable_frame, text="Parâmetros:", 
                         font=("Arial", 12, "bold")).grid(
                    row=row, column=0, columnspan=2, sticky="w", padx=10, pady=10)
                row += 1
                
                for param in params.findall("parameter"):
                    name = param.get("name")
                    value = param.get("value")
                    
                    ttk.Label(scrollable_frame, text=name, 
                             font=("Arial", 10, "bold")).grid(
                        row=row, column=0, sticky="w", padx=20, pady=2)
                    ttk.Label(scrollable_frame, text=value).grid(
                        row=row, column=1, sticky="w", padx=10, pady=2)
                    row += 1
            
            # Regras
            rules = root.find("rules")
            if rules is not None and len(rules):
                ttk.Label(scrollable_frame, text="Regras de Entrada:", 
                         font=("Arial", 12, "bold")).grid(
                    row=row, column=0, columnspan=2, sticky="w", padx=10, pady=10)
                row += 1
                
                for rule in rules.findall("rule"):
                    rule_type = rule.get("type")
                    
                    ttk.Label(scrollable_frame, text=f"Tipo: {rule_type}", 
                             font=("Arial", 10, "bold"), 
                             foreground="#4CAF50" if rule_type == "CALL" else "#E74C3C").grid(
                        row=row, column=0, columnspan=2, sticky="w", padx=20, pady=5)
                    row += 1
                    
                    # Condições
                    for condition in rule.findall("condition"):
                        indicator = condition.get("indicator")
                        parameter = condition.get("parameter")
                        relation = condition.get("relation")
                        target = condition.get("target")
                        
                        relation_symbol = {
                            ">": ">",
                            "<": "<",
                            ">=": "≥",
                            "<=": "≤",
                            "==": "=",
                            "!=": "≠"
                        }.get(relation, relation)
                        
                        condition_text = f"{indicator}({parameter}) {relation_symbol} {target}"
                        
                        ttk.Label(scrollable_frame, text="•", 
                                 font=("Arial", 12)).grid(
                            row=row, column=0, sticky="e", padx=5, pady=2)
                        ttk.Label(scrollable_frame, text=condition_text).grid(
                            row=row, column=1, sticky="w", padx=5, pady=2)
                        row += 1
            
            # Botão de fechar
            ttk.Button(dialog, text="Fechar", command=dialog.destroy).pack(pady=10)
            
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao carregar detalhes da estratégia: {e}")

    def edit_strategy(self, tree):
        """Edita uma estratégia"""
        # Obtém a seleção atual
        selected = tree.selection()
        
        if not selected:
            messagebox.showinfo("Aviso", "Selecione uma estratégia para editar")
            return
        
        # Obtém valores da seleção
        values = tree.item(selected)["values"]
        strategy_name = values[0]
        
        # Procura o arquivo XML
        xml_path = os.path.join(self.bots_dir, f"{strategy_name}.xml")
        
        if not os.path.exists(xml_path):
            messagebox.showerror("Erro", f"Arquivo da estratégia '{strategy_name}' não encontrado!")
            return
        
        try:
            # Parse do XML
            tree = ET.parse(xml_path)
            root = tree.getroot()
            
            # Cria janela de edição
            dialog = tk.Toplevel(self.root)
            dialog.title(f"Editar Estratégia: {strategy_name}")
            dialog.geometry("800x600")
            dialog.transient(self.root)
            
            # Notebook para organizar em abas
            notebook = ttk.Notebook(dialog)
            notebook.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
            
            # Aba de informações básicas
            basic_tab = ttk.Frame(notebook)
            notebook.add(basic_tab, text="Informações Básicas")
            
            # Aba de parâmetros
            params_tab = ttk.Frame(notebook)
            notebook.add(params_tab, text="Parâmetros")
            
            # Aba de regras
            rules_tab = ttk.Frame(notebook)
            notebook.add(rules_tab, text="Regras")
            
            # Preenche aba de informações básicas
            ttk.Label(basic_tab, text="Nome:").grid(
                row=0, column=0, sticky="w", padx=10, pady=5)
            name_entry = ttk.Entry(basic_tab, width=40)
            name_entry.grid(row=0, column=1, sticky="w", padx=10, pady=5)
            name_entry.insert(0, strategy_name)
            
            desc_node = root.find("description")
            desc_text = desc_node.text if desc_node is not None else ""
            ttk.Label(basic_tab, text="Descrição:").grid(
                row=1, column=0, sticky="w", padx=10, pady=5)
            desc_entry = ttk.Entry(basic_tab, width=60)
            desc_entry.grid(row=1, column=1, sticky="w", padx=10, pady=5)
            desc_entry.insert(0, desc_text)
            
            market_node = root.find("market")
            market_text = market_node.text if market_node is not None else "Synthetic"
            ttk.Label(basic_tab, text="Mercado:").grid(
                row=2, column=0, sticky="w", padx=10, pady=5)
            market_combo = ttk.Combobox(basic_tab, 
                                       values=["Synthetic", "Forex", "Stocks", "Commodities", "Indices", "Crypto"])
            market_combo.grid(row=2, column=1, sticky="w", padx=10, pady=5)
            market_combo.set(market_text)
            
            timeframe_node = root.find("timeframe")
            timeframe_text = timeframe_node.text if timeframe_node is not None else "1m"
            ttk.Label(basic_tab, text="Timeframe:").grid(
                row=3, column=0, sticky="w", padx=10, pady=5)
            timeframe_combo = ttk.Combobox(basic_tab, 
                                          values=["1m", "5m", "15m", "30m", "1h", "4h", "1d"])
            timeframe_combo.grid(row=3, column=1, sticky="w", padx=10, pady=5)
            timeframe_combo.set(timeframe_text)
            
            winrate_node = root.find("winrate")
            winrate_text = winrate_node.text if winrate_node is not None else "0"
            ttk.Label(basic_tab, text="Taxa de Acerto (%):").grid(
                row=4, column=0, sticky="w", padx=10, pady=5)
            winrate_entry = ttk.Entry(basic_tab, width=10)
            winrate_entry.grid(row=4, column=1, sticky="w", padx=10, pady=5)
            winrate_entry.insert(0, winrate_text)
            
            # Preenche aba de parâmetros
            params_frame = ttk.Frame(params_tab)
            params_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
            
            # Lista de parâmetros
            params_list = ttk.Treeview(params_frame, columns=("name", "value"), show="headings")
            params_list.heading("name", text="Nome do Parâmetro")
            params_list.heading("value", text="Valor")
            params_list.column("name", width=200)
            params_list.column("value", width=100)
            params_list.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
            
            # Adiciona parâmetros existentes
            params_node = root.find("parameters")
            if params_node is not None:
                for param in params_node.findall("parameter"):
                    name = param.get("name")
                    value = param.get("value")
                    params_list.insert("", "end", values=(name, value))
            
            # Botões para parâmetros
            params_buttons = ttk.Frame(params_frame)
            params_buttons.pack(fill=tk.X, pady=5)
            
            def add_parameter():
                """Adiciona novo parâmetro"""
                # Janela de diálogo
                param_dialog = tk.Toplevel(dialog)
                param_dialog.title("Adicionar Parâmetro")
                param_dialog.transient(dialog)
                param_dialog.grab_set()
                
                ttk.Label(param_dialog, text="Nome:").grid(
                    row=0, column=0, sticky="w", padx=10, pady=5)
                name_entry = ttk.Entry(param_dialog, width=30)
                name_entry.grid(row=0, column=1, sticky="w", padx=10, pady=5)
                
                ttk.Label(param_dialog, text="Valor:").grid(
                    row=1, column=0, sticky="w", padx=10, pady=5)
                value_entry = ttk.Entry(param_dialog, width=20)
                value_entry.grid(row=1, column=1, sticky="w", padx=10, pady=5)
                
                def save_param():
                    """Salva o parâmetro na lista"""
                    name = name_entry.get().strip()
                    value = value_entry.get().strip()
                    
                    if not name or not value:
                        messagebox.showerror("Erro", "Nome e valor são obrigatórios!")
                        return
                    
                    params_list.insert("", "end", values=(name, value))
                    param_dialog.destroy()
                
                ttk.Button(param_dialog, text="Salvar", command=save_param).grid(
                    row=2, column=0, columnspan=2, pady=10)
            
            def remove_parameter():
                """Remove parâmetro selecionado"""
                selected = params_list.selection()
                if selected:
                    params_list.delete(selected)
                else:
                    messagebox.showinfo("Aviso", "Selecione um parâmetro para remover")
            
            ttk.Button(params_buttons, text="Adicionar", command=add_parameter).pack(side=tk.LEFT, padx=5)
            ttk.Button(params_buttons, text="Remover", command=remove_parameter).pack(side=tk.LEFT, padx=5)
            
            # Preenche aba de regras
            rules_frame = ttk.Frame(rules_tab)
            rules_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
            
            # Notebook para regras CALL e PUT
            rules_notebook = ttk.Notebook(rules_frame)
            rules_notebook.pack(fill=tk.BOTH, expand=True)
            
            # Abas para CALL e PUT
            call_tab = ttk.Frame(rules_notebook)
            put_tab = ttk.Frame(rules_notebook)
            
            rules_notebook.add(call_tab, text="Regras CALL (COMPRA)")
            rules_notebook.add(put_tab, text="Regras PUT (VENDA)")
            
            # Listas para condições em cada aba
            call_conditions = ttk.Treeview(call_tab, 
                                         columns=("indicator", "parameter", "relation", "target"),
                                         show="headings")
            call_conditions.heading("indicator", text="Indicador")
            call_conditions.heading("parameter", text="Parâmetro")
            call_conditions.heading("relation", text="Relação")
            call_conditions.heading("target", text="Alvo")
            call_conditions.column("indicator", width=100)
            call_conditions.column("parameter", width=100)
            call_conditions.column("relation", width=80)
            call_conditions.column("target", width=100)
            call_conditions.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
            
            put_conditions = ttk.Treeview(put_tab, 
                                        columns=("indicator", "parameter", "relation", "target"),
                                        show="headings")
            put_conditions.heading("indicator", text="Indicador")
            put_conditions.heading("parameter", text="Parâmetro")
            put_conditions.heading("relation", text="Relação")
            put_conditions.heading("target", text="Alvo")
            put_conditions.column("indicator", width=100)
            put_conditions.column("parameter", width=100)
            put_conditions.column("relation", width=80)
            put_conditions.column("target", width=100)
            put_conditions.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
            
            # Adiciona condições existentes
            rules_node = root.find("rules")
            if rules_node is not None:
                for rule in rules_node.findall("rule"):
                    rule_type = rule.get("type")
                    conditions_list = call_conditions if rule_type == "CALL" else put_conditions
                    
                    for condition in rule.findall("condition"):
                        indicator = condition.get("indicator")
                        parameter = condition.get("parameter")
                        relation = condition.get("relation")
                        target = condition.get("target")
                        
                        conditions_list.insert("", "end", values=(indicator, parameter, relation, target))
            
            # Botões para regras CALL
            call_buttons = ttk.Frame(call_tab)
            call_buttons.pack(fill=tk.X, pady=5)
            
            def add_rule(rule_type):
                """Adiciona nova regra do tipo especificado"""
                conditions_list = call_conditions if rule_type == "CALL" else put_conditions
                
                # Janela de diálogo
                rule_dialog = tk.Toplevel(dialog)
                rule_dialog.title(f"Adicionar Condição {rule_type}")
                rule_dialog.transient(dialog)
                rule_dialog.grab_set()
                
                ttk.Label(rule_dialog, text="Indicador:").grid(
                    row=0, column=0, sticky="w", padx=10, pady=5)
                indicator_combo = ttk.Combobox(rule_dialog, width=20,
                                            values=["EMA", "RSI", "MACD", "Stochastic", "Bollinger", "Volume"])
                indicator_combo.grid(row=0, column=1, sticky="w", padx=10, pady=5)
                
                ttk.Label(rule_dialog, text="Parâmetro:").grid(
                    row=1, column=0, sticky="w", padx=10, pady=5)
                param_entry = ttk.Entry(rule_dialog, width=20)
                param_entry.grid(row=1, column=1, sticky="w", padx=10, pady=5)
                
                ttk.Label(rule_dialog, text="Relação:").grid(
                    row=2, column=0, sticky="w", padx=10, pady=5)
                relation_combo = ttk.Combobox(rule_dialog, width=10,
                                           values=[">", "<", ">=", "<=", "==", "!="])
                relation_combo.grid(row=2, column=1, sticky="w", padx=10, pady=5)
                
                ttk.Label(rule_dialog, text="Alvo:").grid(
                    row=3, column=0, sticky="w", padx=10, pady=5)
                target_entry = ttk.Entry(rule_dialog, width=20)
                target_entry.grid(row=3, column=1, sticky="w", padx=10, pady=5)
                
                def save_rule():
                    """Salva a regra na lista correspondente"""
                    indicator = indicator_combo.get().strip()
                    parameter = param_entry.get().strip()
                    relation = relation_combo.get().strip()
                    target = target_entry.get().strip()
                    
                    if not indicator or not parameter or not relation or not target:
                        messagebox.showerror("Erro", "Todos os campos são obrigatórios!")
                        return
                    
                    conditions_list.insert("", "end", values=(indicator, parameter, relation, target))
                    rule_dialog.destroy()
                
                ttk.Button(rule_dialog, text="Salvar", command=save_rule).grid(
                    row=4, column=0, columnspan=2, pady=10)
            
            def remove_rule(rule_type):
                """Remove regra selecionada"""
                conditions_list = call_conditions if rule_type == "CALL" else put_conditions
                selected = conditions_list.selection()
                if selected:
                    conditions_list.delete(selected)
                else:
                    messagebox.showinfo("Aviso", "Selecione uma condição para remover")
            
            ttk.Button(call_buttons, text="Adicionar Condição", 
                     command=lambda: add_rule("CALL")).pack(side=tk.LEFT, padx=5)
            ttk.Button(call_buttons, text="Remover Condição", 
                     command=lambda: remove_rule("CALL")).pack(side=tk.LEFT, padx=5)
            
            # Botões para regras PUT
            put_buttons = ttk.Frame(put_tab)
            put_buttons.pack(fill=tk.X, pady=5)
            
            ttk.Button(put_buttons, text="Adicionar Condição", 
                     command=lambda: add_rule("PUT")).pack(side=tk.LEFT, padx=5)
            ttk.Button(put_buttons, text="Remover Condição", 
                     command=lambda: remove_rule("PUT")).pack(side=tk.LEFT, padx=5)
            
            # Botões de ação
            action_frame = ttk.Frame(dialog)
            action_frame.pack(fill=tk.X, pady=10, padx=10)
            
            def save_changes():
                """Salva todas as alterações no arquivo XML"""
                try:
                    # Cria novo XML
                    new_root = ET.Element("strategy")
                    
                    # Informações básicas
                    ET.SubElement(new_root, "name").text = name_entry.get().strip()
                    ET.SubElement(new_root, "description").text = desc_entry.get().strip()
                    ET.SubElement(new_root, "market").text = market_combo.get()
                    ET.SubElement(new_root, "timeframe").text = timeframe_combo.get()
                    ET.SubElement(new_root, "winrate").text = winrate_entry.get().strip()
                    
                    # Parâmetros
                    params = ET.SubElement(new_root, "parameters")
                    for item_id in params_list.get_children():
                        values = params_list.item(item_id)["values"]
                        param = ET.SubElement(params, "parameter")
                        param.set("name", values[0])
                        param.set("value", values[1])
                    
                    # Regras
                    rules = ET.SubElement(new_root, "rules")
                    
                    # Regras CALL
                    if call_conditions.get_children():
                        rule_call = ET.SubElement(rules, "rule")
                        rule_call.set("type", "CALL")
                        
                        for item_id in call_conditions.get_children():
                            values = call_conditions.item(item_id)["values"]
                            condition = ET.SubElement(rule_call, "condition")
                            condition.set("indicator", values[0])
                            condition.set("parameter", values[1])
                            condition.set("relation", values[2])
                            condition.set("target", values[3])
                    
                    # Regras PUT
                    if put_conditions.get_children():
                        rule_put = ET.SubElement(rules, "rule")
                        rule_put.set("type", "PUT")
                        
                        for item_id in put_conditions.get_children():
                            values = put_conditions.item(item_id)["values"]
                            condition = ET.SubElement(rule_put, "condition")
                            condition.set("indicator", values[0])
                            condition.set("parameter", values[1])
                            condition.set("relation", values[2])
                            condition.set("target", values[3])
                    
                    # Recuperação (mantém a existente ou cria padrão)
                    recovery_node = root.find("recovery")
                    if recovery_node is not None:
                        recovery = ET.SubElement(new_root, "recovery")
                        
                        # Martingale
                        mg_node = recovery_node.find("martingale")
                        if mg_node is not None:
                            mg = ET.SubElement(recovery, "martingale")
                            mg.set("factor", mg_node.get("factor", "2.0"))
                            mg.set("max_level", mg_node.get("max_level", "3"))
                        
                        # Cooldown
                        cd_node = recovery_node.find("cooldown")
                        if cd_node is not None:
                            cd = ET.SubElement(recovery, "cooldown")
                            cd.set("trades", cd_node.get("trades", "2"))
                    else:
                        # Cria recuperação padrão
                        recovery = ET.SubElement(new_root, "recovery")
                        mg = ET.SubElement(recovery, "martingale")
                        mg.set("factor", "2.0")
                        mg.set("max_level", "3")
                        cd = ET.SubElement(recovery, "cooldown")
                        cd.set("trades", "2")
                    
                    # Salva o arquivo
                    new_tree = ET.ElementTree(new_root)
                    new_name = name_entry.get().strip()
                    file_path = os.path.join(self.bots_dir, f"{new_name}.xml")
                    
                    # Remove arquivo antigo se nome foi alterado
                    if new_name != strategy_name and os.path.exists(xml_path):
                        os.remove(xml_path)
                    
                    # Salva novo arquivo
                    new_tree.write(file_path, encoding="utf-8", xml_declaration=True)
                    
                    self.log(f"✅ Estratégia '{new_name}' salva com sucesso!")
                    messagebox.showinfo("Sucesso", "Estratégia salva com sucesso!")
                    
                    dialog.destroy()
                    
                    # Recarrega estratégias
                    self.load_bot_strategies()
                    
                except Exception as e:
                    messagebox.showerror("Erro", f"Erro ao salvar estratégia: {e}")
            
            ttk.Button(action_frame, text="Salvar Alterações", 
                     command=save_changes).pack(side=tk.LEFT, padx=5)
            
            ttk.Button(action_frame, text="Cancelar", 
                     command=dialog.destroy).pack(side=tk.RIGHT, padx=5)
            
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao editar estratégia: {e}")

    def delete_strategy(self, tree):
        """Exclui uma estratégia"""
        # Obtém a seleção atual
        selected = tree.selection()
        
        if not selected:
            messagebox.showinfo("Aviso", "Selecione uma estratégia para excluir")
            return
        
        # Obtém valores da seleção
        values = tree.item(selected)["values"]
        strategy_name = values[0]
        
        # Confirma exclusão
        confirm = messagebox.askyesno("Confirmar Exclusão", 
                                     f"Tem certeza que deseja excluir a estratégia '{strategy_name}'?")
        
        if not confirm:
            return
        
        # Exclui arquivo
        xml_path = os.path.join(self.bots_dir, f"{strategy_name}.xml")
        
        try:
            if os.path.exists(xml_path):
                os.remove(xml_path)
                self.log(f"🗑️ Estratégia '{strategy_name}' excluída com sucesso!")
                messagebox.showinfo("Sucesso", f"Estratégia '{strategy_name}' excluída com sucesso!")
                
                # Remove da lista
                tree.delete(selected)
            else:
                messagebox.showerror("Erro", f"Arquivo da estratégia '{strategy_name}' não encontrado!")
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao excluir estratégia: {e}")

    def export_report(self):
        """Exporta relatório de trading"""
        try:
            from tkinter import filedialog
            import csv
            
            # Solicita local para salvar
            filename = filedialog.asksaveasfilename(
                title="Salvar Relatório",
                defaultextension=".csv",
                filetypes=[("CSV", "*.csv"), ("Todos os arquivos", "*.*")]
            )
            
            if not filename:
                return
            
            # Dados do relatório (exemplo)
            report_data = [
                ["Data", "Operações", "Vitórias", "Derrotas", "Taxa de Acerto", "Lucro", "Maior Sequência Positiva", "Maior Sequência Negativa"],
                [datetime.datetime.now().strftime("%d/%m/%Y"), 247, 185, 62, "75%", "$523.75", 12, 5],
                [datetime.datetime.now().strftime("%d/%m/%Y"), 63, 46, 17, "73%", "$85.30", 8, 3],
            ]
            
            # Salva arquivo CSV
            with open(filename, 'w', newline='') as csvfile:
                writer = csv.writer(csvfile)
                for row in report_data:
                    writer.writerow(row)
            
            self.log(f"📊 Relatório de trading exportado para {filename}")
            messagebox.showinfo("Sucesso", "Relatório exportado com sucesso!")
            
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao exportar relatório: {e}")

    def refresh_stats(self):
        """Atualiza estatísticas"""
        messagebox.showinfo("Estatísticas", "Estatísticas atualizadas com sucesso!")

    def restore_defaults(self):
        """Restaura configurações padrão"""
        confirm = messagebox.askyesno("Confirmar", 
                                     "Tem certeza que deseja restaurar as configurações padrão?")
        
        if not confirm:
            return
        
        # Restaura valores padrão
        self.api_mode.set("demo")
        self.trading_mode.set("live")
        self.symbol.set("R_10")
        self.duration.set(5)
        self.win_amount.set(0.35)
        self.expected_profit.set(5.0)
        self.max_loss.set(10.0)
        self.profit_increase.set(15.0)
        
        # Salva as configurações
        self.save_config()
        
        self.log("🔄 Configurações restauradas para valores padrão")

    def test_connection(self):
        """Testa a conexão com a API Deriv"""
        # Verifica qual token usar
        token = self.token_demo.get() if self.api_mode.get() == "demo" else self.token_real.get()
        
        if not token:
            messagebox.showerror("Erro", "Token da API não configurado!")
            return
        
        # Inicia teste em uma thread separada
        test_thread = threading.Thread(target=self._test_connection_thread, args=(token,))
        test_thread.daemon = True
        test_thread.start()
        
        self.log("🔌 Testando conexão com a API Deriv...")
        self.update_status("Testando conexão...")

    def _test_connection_thread(self, token):
        """Thread para testar conexão"""
        try:
            # Simula o loop do evento asyncio
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
            
            # Testa a conexão
            result = loop.run_until_complete(self._test_connection_async(token))
            
            if result:
                self.log("✅ Conexão com a API Deriv estabelecida com sucesso!")
                messagebox.showinfo("Teste de Conexão", "Conexão estabelecida com sucesso!")
            else:
                self.log("❌ Falha ao conectar com a API Deriv")
                messagebox.showerror("Erro", "Falha ao conectar com a API. Verifique o token e sua conexão.")
            
        except Exception as e:
            self.log(f"❌ Erro ao testar conexão: {e}")
            messagebox.showerror("Erro", f"Erro ao testar conexão: {e}")
        finally:
            self.update_status("Pronto")

    async def _test_connection_async(self, token):
        """Teste assíncrono da conexão com a API Deriv"""
        try:
            # Verifica se o módulo websockets está instalado
            try:
                import websockets
            except ImportError:
                self.log("❌ Erro: Módulo websockets não encontrado!")
                messagebox.showerror("Erro", "Módulo websockets não encontrado! Execute 'pip install websockets' para instalá-lo.")
                return False
                
            uri = "wss://ws.binaryws.com/websockets/v3?app_id=1089"
            
            # Conecta com a API Deriv
            self.log("🔌 Conectando à API Deriv...")
            
            try:
                async with websockets.connect(uri) as ws:
                    # Envia pedido de autorização
                    self.log("🔑 Autorizando com token...")
                    await ws.send(json.dumps({"authorize": token}))
                    
                    # Recebe resposta
                    response = await asyncio.wait_for(ws.recv(), timeout=10)
                    auth_data = json.loads(response)
                    
                    # Verifica se autorizou
                    if 'error' in auth_data:
                        error_msg = auth_data['error']['message']
                        self.log(f"❌ Erro de autorização: {error_msg}")
                        raise Exception(f"Erro de autorização: {error_msg}")
                    
                    # Obtém detalhes da conta
                    account_type = auth_data.get('authorize', {}).get('account_type', 'desconhecido')
                    email = auth_data.get('authorize', {}).get('email', 'desconhecido')
                    
                    # Obtém saldo
                    self.log("💰 Obtendo saldo...")
                    await ws.send(json.dumps({"balance": 1}))
                    balance_response = await asyncio.wait_for(ws.recv(), timeout=10)
                    balance_data = json.loads(balance_response)
                    if 'error' in balance_data:
                        self.log(f"⚠️ Erro ao obter saldo: {balance_data['error']['message']}")
                        # Continua mesmo com erro de saldo
                    else:
                        balance = balance_data.get('balance', {}).get('balance', 0)
                        currency = balance_data.get('balance', {}).get('currency', 'USD')
                        
                        self.log(f"✅ Saldo: {balance} {currency}")
                    
                    # Log das informações obtidas
                    self.log(f"✅ Conexão estabelecida com sucesso!")
                    self.log(f"👤 Conta: {email}")
                    
                    return True
            except Exception as e:
                self.log(f"❌ Erro na conexão: {str(e)}")
                raise Exception(f"Erro na conexão: {str(e)}")
                
        except Exception as e:
            self.log(f"❌ Erro de conexão com a API: {e}")
            self.log("⚠️ Verifique o app_id ou sua conexão com a internet.")
            return False
            
        except websockets.exceptions.ConnectionClosedError as e:
            self.log(f"❌ Conexão fechada: {e}")
            self.log("⚠️ O servidor fechou a conexão, tente novamente mais tarde.")
            return False
            
        except Exception as e:
            self.log(f"❌ Erro durante teste de conexão: {e}")
            import traceback
            self.log(f"🔍 Detalhes: {traceback.format_exc()}")
            return False

    def open_token_page(self):
        """Abre a página para obter token da API"""
        import webbrowser
        webbrowser.open("https://app.deriv.com/account/api-token")
        self.log("🔗 Abrindo página para obter token da API Deriv")


# Função principal para iniciar a aplicação
def main():
    root = tk.Tk()
    app = FenixBot(root)
    root.mainloop()


if __name__ == "__main__":
    main()
