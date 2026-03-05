# meu-backup
using Microsoft.Win32;
using System;
using System.IO;
using System.Windows.Forms;

namespace SistemaManutencao
{
    /// <summary>
    /// Classe para gerenciar inicialização automática do sistema com o Windows
    /// Esta é a ÚNICA classe de inicialização - não use InicializacaoAutomatica ou SistemaAutoStart
    /// </summary>
    public static class GerenciadorInicializacao
    {
        private const string NOME_APLICACAO = "SistemaManutencao";
        private const string CHAVE_REGISTRO = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Run";

        /// <summary>
        /// Verifica se a aplicação está configurada para iniciar com o Windows
        /// </summary>
        public static bool EstaConfigurado()
        {
            try
            {
                using (RegistryKey key = Registry.CurrentUser.OpenSubKey(CHAVE_REGISTRO, false))
                {
                    if (key != null)
                    {
                        object valor = key.GetValue(NOME_APLICACAO);
                        return valor != null;
                    }
                    return false;
                }
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Erro ao verificar inicializacao: {ex.Message}");
                return false;
            }
        }

        /// <summary>
        /// Habilita a inicialização automática com o Windows
        /// </summary>
        public static bool Habilitar()
        {
            try
            {
                // Obter caminho do executável
                string caminhoExe = Application.ExecutablePath;
                string pastaExe = Path.GetDirectoryName(caminhoExe);

                // Caminho do script VBS (invisível)
                string caminhoVbs = Path.Combine(pastaExe, "IniciarSistema.vbs");

                // Verificar se o VBS existe
                if (!File.Exists(caminhoVbs))
                {
                    // Se não existir, criar o VBS
                    CriarScripts(pastaExe);
                }

                System.Diagnostics.Debug.WriteLine($"Habilitando inicializacao automatica");
                System.Diagnostics.Debug.WriteLine($"Script VBS: {caminhoVbs}");

                using (RegistryKey key = Registry.CurrentUser.OpenSubKey(CHAVE_REGISTRO, true))
                {
                    if (key != null)
                    {
                        // Registrar o VBS (que executa o BAT invisível)
                        key.SetValue(NOME_APLICACAO, $"wscript.exe \"{caminhoVbs}\"");
                        System.Diagnostics.Debug.WriteLine($"Inicializacao automatica habilitada");
                        return true;
                    }
                    else
                    {
                        System.Diagnostics.Debug.WriteLine($"Nao foi possivel abrir chave de registro");
                        return false;
                    }
                }
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Erro ao habilitar: {ex.Message}");
                MessageBox.Show(
                    $"Erro ao habilitar inicializacao automatica:\n\n{ex.Message}\n\n" +
                    "Voce pode precisar de permissoes de administrador.",
                    "Erro",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error
                );
                return false;
            }
        }

        /// <summary>
        /// Desabilita a inicialização automática com o Windows
        /// </summary>
        public static bool Desabilitar()
        {
            try
            {
                System.Diagnostics.Debug.WriteLine($"Desabilitando inicializacao automatica");

                using (RegistryKey key = Registry.CurrentUser.OpenSubKey(CHAVE_REGISTRO, true))
                {
                    if (key != null)
                    {
                        object valor = key.GetValue(NOME_APLICACAO);
                        if (valor != null)
                        {
                            key.DeleteValue(NOME_APLICACAO);
                            System.Diagnostics.Debug.WriteLine($"Inicializacao automatica desabilitada");
                        }
                        return true;
                    }
                    return false;
                }
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Erro ao desabilitar: {ex.Message}");
                MessageBox.Show(
                    $"Erro ao desabilitar inicializacao automatica:\n\n{ex.Message}",
                    "Erro",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error
                );
                return false;
            }
        }

        /// <summary>
        /// Cria os scripts BAT e VBS na pasta do executável
        /// </summary>
        private static void CriarScripts(string pastaDestino)
        {
            try
            {
                // Criar IniciarSistema.bat
                string caminhoBat = Path.Combine(pastaDestino, "IniciarSistema.bat");
                string conteudoBat = @"@echo off
REM INICIAR XAMPP E SISTEMA

REM Iniciar Apache
tasklist /FI ""IMAGENAME eq httpd.exe"" 2>NUL | find /I /N ""httpd.exe"">NUL
if %errorlevel% neq 0 (
    start """" ""C:\xampp\apache_start.bat""
    timeout /t 3 /nobreak >nul
)

REM Iniciar MySQL
tasklist /FI ""IMAGENAME eq mysqld.exe"" 2>NUL | find /I /N ""mysqld.exe"">NUL
if %errorlevel% neq 0 (
    start """" ""C:\xampp\mysql_start.bat""
    timeout /t 5 /nobreak >nul
)

REM Aguardar estabilizar
timeout /t 3 /nobreak >nul

REM Iniciar Sistema
start """" ""%~dp0SistemaManutencao.exe""

exit
";

                File.WriteAllText(caminhoBat, conteudoBat);
                System.Diagnostics.Debug.WriteLine($"BAT criado: {caminhoBat}");

                // Criar IniciarSistema.vbs
                string caminhoVbs = Path.Combine(pastaDestino, "IniciarSistema.vbs");
                string conteudoVbs = @"Set WshShell = CreateObject(""WScript.Shell"")
WshShell.Run chr(34) & WScript.ScriptFullName & ""\..\"" & ""IniciarSistema.bat"" & Chr(34), 0
Set WshShell = Nothing
";

                File.WriteAllText(caminhoVbs, conteudoVbs);
                System.Diagnostics.Debug.WriteLine($"VBS criado: {caminhoVbs}");
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Erro ao criar scripts: {ex.Message}");
                throw;
            }
        }

        /// <summary>
        /// Obtém o caminho configurado no registro
        /// </summary>
        public static string ObterCaminhoConfigurado()
        {
            try
            {
                using (RegistryKey key = Registry.CurrentUser.OpenSubKey(CHAVE_REGISTRO, false))
                {
                    if (key != null)
                    {
                        object valor = key.GetValue(NOME_APLICACAO);
                        return valor?.ToString() ?? "";
                    }
                    return "";
                }
            }
            catch
            {
                return "";
            }
        }
    }
}

using System;
using System.Windows.Forms;

namespace SistemaManutencao
{
    static class Program
    {
        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);

            // IMPORTANTE: Enviar resumo ANTES do login para não bloquear automação
            // O sistema continua funcionando em segundo plano mesmo sem login
            try
            {
                // Executar de forma assíncrona sem bloquear
                System.Threading.Tasks.Task.Run(async () =>
                {
                    await ServicoMonitoramento.VerificarEEnviarResumo();
                });
            }
            catch
            {
                // Ignorar erros para não impedir abertura do sistema
            }

            //Loop de Login/Sistema
            while (true)
            {
                // Mostrar tela de login
                FrmLogin frmLogin = new FrmLogin();
                   DialogResult resultLogin = frmLogin.ShowDialog();

                // Se fechou o login (clicou no X), sair completamente
                     if (resultLogin == DialogResult.Cancel || resultLogin == DialogResult.Abort)
                {
                    break;
                }

                // Se pulou o login ou fez login com sucesso
                        if (resultLogin == DialogResult.OK)
                {
                    // Abrir sistema principal
                    FrmPrincipal frmPrincipal = new FrmPrincipal();
                    Application.Run(frmPrincipal);


                    // Quando fechar o FrmPrincipal, verificar se deve voltar ao login ou sair
                    // Se o DialogResult for Retry, significa que o usuário fez logout e quer voltar ao login
                    if (frmPrincipal.DialogResult == DialogResult.Retry)
                    {
                        // Volta pro início do loop (mostra login novamente)
                        continue;
                    }
                    else
                    {
                        // Se não é Retry, o usuário fechou o sistema - sair completamente
                       break;
                    }
                }
            }
        }
                //else
               // {
                     //ualquer outro resultado, sair;
               // }
            
    }
        

}
using System;
using System.Data;
using System.Threading.Tasks;
using System.Text;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public static class ServicoMonitoramento
    {
        /// <summary>
        /// Verifica e envia manutenções SEMPRE que o sistema abrir
        /// Ordem: 1-HOJE, 2-ATRASADO, 3-PLANEJADO
        /// </summary>
        public static async Task VerificarEEnviarResumo()
        {
            try
            {
                // ATUALIZAR TODAS AS SITUAÇÕES CORRETAMENTE
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // 1. HOJE
                    string queryHoje = @"
                        UPDATE manutencoes 
                        SET situacao = 'HOJE' 
                        WHERE DATE(data_planejada) = CURDATE() 
                        AND situacao NOT IN ('REALIZADO', 'CANCELADO')";
                    
                    MySqlCommand cmdHoje = new MySqlCommand(queryHoje, conn);
                    cmdHoje.ExecuteNonQuery();

                    // 2. ATRASADO
                    string queryAtrasado = @"
                        UPDATE manutencoes 
                        SET situacao = 'ATRASADO' 
                        WHERE DATE(data_planejada) < CURDATE() 
                        AND situacao NOT IN ('REALIZADO', 'CANCELADO')";
                    
                    MySqlCommand cmdAtrasado = new MySqlCommand(queryAtrasado, conn);
                    cmdAtrasado.ExecuteNonQuery();

                    // 3. PLANEJADO
                    string queryPlanejado = @"
                        UPDATE manutencoes 
                        SET situacao = 'PLANEJADO' 
                        WHERE DATE(data_planejada) > CURDATE() 
                        AND situacao NOT IN ('REALIZADO', 'CANCELADO')";
                    
                    MySqlCommand cmdPlanejado = new MySqlCommand(queryPlanejado, conn);
                    cmdPlanejado.ExecuteNonQuery();
                }

                // Buscar manutenções: HOJE + ATRASADAS
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            m.situacao,
                            p.nome_peca,
                            maq.nome_maquina,
                            m.planta,
                            m.data_planejada
                        FROM manutencoes m
                        INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
                        INNER JOIN pecas p ON m.id_peca = p.id_peca
                        WHERE m.situacao IN ('HOJE', 'ATRASADO')
                        ORDER BY 
                            CASE m.situacao
                                WHEN 'HOJE' THEN 1
                                WHEN 'ATRASADO' THEN 2
                            END,
                            m.data_planejada";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    if (dt.Rows.Count == 0)
                    {
                        return; // Sem manutenções, não envia nada
                    }

                    // Montar mensagem SEM EMOJIS
                    StringBuilder sb = new StringBuilder();

                    // Separar por situação
                    DataRow[] hoje = dt.Select("situacao = 'HOJE'");
                    DataRow[] atrasadas = dt.Select("situacao = 'ATRASADO'");

                    // HOJE primeiro (sempre mostrar o cabeçalho)
                    sb.AppendLine("HOJE");
                    if (hoje.Length > 0)
                    {
                        foreach (DataRow row in hoje)
                        {
                            string peca = row["nome_peca"].ToString().ToUpper();
                            string maquina = row["nome_maquina"].ToString().ToUpper();
                            string planta = row["planta"].ToString().ToUpper();

                            sb.AppendLine($"   Trocar {peca}");
                            sb.AppendLine($"   Maquina: {maquina}");
                            sb.AppendLine($"   Planta: {planta}");
                            sb.AppendLine("   ━━━━━━━━━━━━━━━━");
                        }
                    }
                    else
                    {
                        sb.AppendLine("   Sem manutencoes para hoje");
                        sb.AppendLine("   ━━━━━━━━━━━━━━━━");
                    }

                    // ATRASADO depois (sempre mostrar o cabeçalho)
                    sb.AppendLine();
                    sb.AppendLine("ATRASADO");
                    if (atrasadas.Length > 0)
                    {
                        foreach (DataRow row in atrasadas)
                        {
                            string peca = row["nome_peca"].ToString().ToUpper();
                            string maquina = row["nome_maquina"].ToString().ToUpper();
                            string planta = row["planta"].ToString().ToUpper();
                            DateTime dataPlanjada = Convert.ToDateTime(row["data_planejada"]);
                            int diasAtraso = (DateTime.Now.Date - dataPlanjada.Date).Days;

                            sb.AppendLine($"   Trocar {peca}");
                            sb.AppendLine($"   Maquina: {maquina}");
                            sb.AppendLine($"   Planta: {planta}");
                            sb.AppendLine($"   Atrasado: {diasAtraso} dia(s)");
                            sb.AppendLine("   ━━━━━━━━━━━━━━━━");
                        }
                    }
                    else
                    {
                        sb.AppendLine("   Nenhuma manutencao atrasada");
                    }

                    string mensagem = sb.ToString();

                    // Adicionar assinatura
                    sb.AppendLine();
                    sb.AppendLine("Mensagem automatica do Sistema de Manutencao");

                    // ENVIAR WHATSAPP
                    await Task.Run(() =>
                    {
                        try
                        {
                            // Obter número do grupo/contato configurado
                            string numeroGrupo = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "");
                            
                            // VALIDAR NÚMERO
                            if (string.IsNullOrWhiteSpace(numeroGrupo))
                            {
                                System.Diagnostics.Debug.WriteLine("❌ ERRO: Número do WhatsApp não configurado!");
                                System.Diagnostics.Debug.WriteLine("Execute o SQL: configurar_whatsapp.sql");
                                
                                // Mostrar erro mas não travar o sistema
                                System.Windows.Forms.MessageBox.Show(
                                    "⚠️ Número do WhatsApp não configurado!\n\n" +
                                    "Para ativar o envio automático:\n" +
                                    "1. Execute o arquivo: configurar_whatsapp.sql\n" +
                                    "2. Configure o número do grupo/contato\n\n" +
                                    "O sistema continuará funcionando normalmente.",
                                    "Configuração WhatsApp",
                                    System.Windows.Forms.MessageBoxButtons.OK,
                                    System.Windows.Forms.MessageBoxIcon.Warning
                                );
                                return;
                            }
                            
                            // Enviar mensagem via WhatsApp
                            ServicoWhatsApp.EnviarMensagem(numeroGrupo, sb.ToString());
                            
                            // Log de sucesso
                            System.Diagnostics.Debug.WriteLine("=== RESUMO ENVIADO VIA WHATSAPP ===");
                            System.Diagnostics.Debug.WriteLine($"Número: {numeroGrupo}");
                            System.Diagnostics.Debug.WriteLine(sb.ToString());
                            System.Diagnostics.Debug.WriteLine("===================================");
                        }
                        catch (Exception ex)
                        {
                            System.Diagnostics.Debug.WriteLine($"❌ Erro ao enviar WhatsApp: {ex.Message}");
                        }
                    });
                }
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"Erro no monitoramento: {ex.Message}");
            }
        }
    }
}

using System;
using System.Diagnostics;
using System.IO;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace SistemaManutencao
{
    /// <summary>
    /// Serviço simplificado de WhatsApp usando apenas WhatsApp Web
    /// VERSÃO OTIMIZADA
    /// </summary>
    public static class ServicoWhatsApp
    {
        private static readonly string pastaLogs = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "logs");

        // =====================================================
        // MÉTODO: INICIALIZAR LOGS
        // =====================================================
        private static void InicializarLogs()
        {
            Directory.CreateDirectory(pastaLogs);
        }

        // =====================================================
        // MÉTODO: REGISTRAR LOG
        // =====================================================
        private static void RegistrarLog(string mensagem)
        {
            try
            {
                InicializarLogs();
                string dataHora = DateTime.Now.ToString("dd/MM/yyyy HH:mm:ss");
                string linha = $"[{dataHora}] {mensagem}";

                // Console de debug
                Debug.WriteLine(linha);

                // Arquivo de log
                string nomeArquivo = $"whatsapp_{DateTime.Now:yyyyMMdd}.txt";
                string caminhoArquivo = Path.Combine(pastaLogs, nomeArquivo);
                File.AppendAllText(caminhoArquivo, linha + Environment.NewLine);
            }
            catch
            {
                // Ignorar erros de log
            }
        }

        // =====================================================
        // MÉTODO: ENCONTRAR CAMINHO DO CHROME
        // =====================================================
        private static string EncontrarChrome()
        {
            RegistrarLog("🔍 Procurando Chrome...");

            string[] caminhosPossiveis = {
                @"C:\Program Files\Google\Chrome\Application\chrome.exe",
                @"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe",
                Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                    @"Google\Chrome\Application\chrome.exe"),
                Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ProgramFiles),
                    @"Google\Chrome\Application\chrome.exe")
            };

            foreach (string caminho in caminhosPossiveis)
            {
                RegistrarLog($"   Verificando: {caminho}");
                if (File.Exists(caminho))
                {
                    RegistrarLog($"✅ Chrome encontrado: {caminho}");
                    return caminho;
                }
            }

            RegistrarLog("❌ Chrome NÃO encontrado em nenhum caminho padrão");
            return null;
        }

        // =====================================================
        // MÉTODO: ABRIR URL NO CHROME
        // =====================================================
        private static void AbrirNoChrome(string url)
        {
            try
            {
                RegistrarLog("🌐 Tentando abrir URL no Chrome...");
                RegistrarLog($"   URL: {url.Substring(0, Math.Min(100, url.Length))}...");

                string chromeExe = EncontrarChrome();

                if (chromeExe != null)
                {
                    RegistrarLog($"📂 Executável do Chrome: {chromeExe}");

                    // Tentar abrir com Process.Start
                    ProcessStartInfo startInfo = new ProcessStartInfo
                    {
                        FileName = chromeExe,
                        Arguments = $"\"{url}\"",
                        UseShellExecute = false
                    };

                    RegistrarLog($"⚙️ Comando: {startInfo.FileName}");
                    RegistrarLog($"⚙️ Argumentos: {startInfo.Arguments}");

                    Process processo = Process.Start(startInfo);

                    if (processo != null)
                    {
                        RegistrarLog($"✅ Chrome iniciado! PID: {processo.Id}");
                    }
                    else
                    {
                        RegistrarLog("⚠️ Process.Start retornou null");
                    }
                }
                else
                {
                    RegistrarLog("⚠️ Chrome não encontrado, tentando navegador padrão...");
                    
                    // Fallback: abrir no navegador padrão
                    ProcessStartInfo startInfo = new ProcessStartInfo
                    {
                        FileName = url,
                        UseShellExecute = true
                    };

                    Process processo = Process.Start(startInfo);
                    
                    if (processo != null)
                    {
                        RegistrarLog($"✅ Navegador padrão iniciado! PID: {processo.Id}");
                    }
                    else
                    {
                        RegistrarLog("❌ Falha ao abrir navegador padrão");
                    }
                }
            }
            catch (Exception ex)
            {
                RegistrarLog($"❌ EXCEÇÃO ao abrir navegador: {ex.Message}");
                RegistrarLog($"   Stack: {ex.StackTrace}");
                throw;
            }
        }

        // =====================================================
        // MÉTODO: ENVIAR MENSAGEM VIA WHATSAPP WEB
        // =====================================================
        public static void EnviarMensagem(string numeroGrupo, string mensagem)
        {
            try
            {
                RegistrarLog("========================================");
                RegistrarLog("📱 INICIANDO ENVIO DE MENSAGEM");
                RegistrarLog("========================================");
                RegistrarLog($"📱 Número recebido: {numeroGrupo}");
                RegistrarLog($"📝 Tamanho da mensagem: {mensagem.Length} caracteres");

                // Limpar número (remover caracteres especiais)
                string numeroLimpo = LimparNumero(numeroGrupo);

                RegistrarLog($"🧹 Número limpo: {numeroLimpo}");

                // Validar número
                ValidarNumero(numeroLimpo);

                // Montar mensagem formatada para URL
                string mensagemUrl = FormatarMensagemParaUrl(mensagem);

                // Montar URL completa
                string url = $"https://web.whatsapp.com/send?phone={numeroLimpo}&text={mensagemUrl}";

                RegistrarLog($"🌐 URL completa montada (tamanho: {url.Length} caracteres)");

                // Abrir no Chrome
                RegistrarLog("🚀 Abrindo navegador...");
                AbrirNoChrome(url);

                RegistrarLog("⏰ Aguardando WhatsApp carregar (25 segundos)...");
                System.Threading.Thread.Sleep(25000); // Aguarda WhatsApp Web carregar

                RegistrarLog("⌨️ Enviando automação (Enter direto)...");
                
                // Pressionar Enter para enviar (sem Tab)
                SendKeys.SendWait("{ENTER}");

                RegistrarLog("✅ AUTOMAÇÃO CONCLUÍDA");
                RegistrarLog("========================================");
                
                MessageBox.Show(
                    "✅ WhatsApp Web aberto!\n\n" +
                    "📱 Sistema aguardou 25 segundos\n" +
                    "⌨️ Enter enviado automaticamente\n\n" +
                    "⚠️ Se não enviou:\n" +
                    "• Faça login no WhatsApp Web\n" +
                    "• Verifique se carregou corretamente\n" +
                    "• Clique no campo de mensagem\n" +
                    "• Pressione Enter manualmente\n\n" +
                    $"📂 Logs em: {pastaLogs}",
                    "WhatsApp",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Information
                );
            }
            catch (Exception ex)
            {
                RegistrarLog($"❌ ERRO AO ENVIAR: {ex.Message}");
                RegistrarLog($"   Stack: {ex.StackTrace}");
                
                MessageBox.Show(
                    $"❌ Erro ao enviar mensagem:\n\n{ex.Message}\n\n" +
                    $"Logs salvos em: {pastaLogs}",
                    "Erro",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error
                );
                
                throw;
            }
        }

        // =====================================================
        // FUNÇÕES LOCAIS AUXILIARES
        // =====================================================
        
        private static string LimparNumero(string numero)
        {
            return numero
                .Replace("-", "")
                .Replace(" ", "")
                .Replace("(", "")
                .Replace(")", "")
                .Replace("+", "");
        }

        private static void ValidarNumero(string numeroLimpo)
        {
            if (string.IsNullOrWhiteSpace(numeroLimpo))
            {
                RegistrarLog("❌ ERRO: Número vazio após limpeza!");
                throw new Exception("Número do WhatsApp está vazio!");
            }

            if (numeroLimpo.Length < 10)
            {
                RegistrarLog($"⚠️ AVISO: Número muito curto ({numeroLimpo.Length} dígitos)");
            }
        }

        private static string FormatarMensagemParaUrl(string mensagem)
        {
            RegistrarLog("🔄 Formatando mensagem para URL...");
            
            string mensagemUrl = mensagem
                .Replace("\r\n", "%0A")
                .Replace("\n", "%0A")
                .Replace("\r", "%0A");

            // Escapar caracteres especiais
            mensagemUrl = Uri.EscapeDataString(mensagemUrl)
                .Replace("%250A", "%0A");

            RegistrarLog($"✅ Mensagem formatada: {mensagemUrl.Substring(0, Math.Min(50, mensagemUrl.Length))}...");
            
            return mensagemUrl;
        }

        // =====================================================
        // MÉTODO: TESTE RÁPIDO
        // =====================================================
        public static void TesteRapido()
        {
            try
            {
                RegistrarLog("========================================");
                RegistrarLog("🧪 INICIANDO TESTE RÁPIDO");
                RegistrarLog("========================================");

                string numeroGrupo = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "5519989102583");
                RegistrarLog($"📱 Número obtido do banco: {numeroGrupo}");

                if (string.IsNullOrEmpty(numeroGrupo))
                {
                    RegistrarLog("⚠️ Número vazio, usando padrão");
                    numeroGrupo = "5519989102583";
                }

                string mensagemTeste = $"🧪 TESTE DO SISTEMA\n\n" +
                                      $"✅ WhatsApp configurado!\n" +
                                      $"📅 {DateTime.Now:dd/MM/yyyy HH:mm:ss}\n\n" +
                                      $"_Mensagem automática de teste_";

                RegistrarLog($"📝 Mensagem de teste criada ({mensagemTeste.Length} caracteres)");

                EnviarMensagem(numeroGrupo, mensagemTeste);
                
                RegistrarLog("✅ Teste concluído");
            }
            catch (Exception ex)
            {
                RegistrarLog($"❌ Erro no teste: {ex.Message}");
                
                MessageBox.Show(
                    $"❌ Erro ao testar:\n\n{ex.Message}\n\n" +
                    $"Logs salvos em: {pastaLogs}",
                    "Erro",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error
                );
            }
        }

        // =====================================================
        // MÉTODO: TESTAR CONEXÃO
        // =====================================================
        public static async Task<bool> TestarConexao()
        {
            return await Task.Run(() =>
            {
                try
                {
                    RegistrarLog("🔍 Testando conexão...");

                    // Verificar Chrome
                    string chrome = EncontrarChrome();
                    if (chrome == null)
                    {
                        RegistrarLog("❌ Chrome não encontrado");
                        MessageBox.Show(
                            "⚠️ Google Chrome não encontrado!\n\n" +
                            "Instale o Google Chrome:\n" +
                            "https://www.google.com/chrome\n\n" +
                            "Ou o sistema tentará usar o navegador padrão.",
                            "Chrome Recomendado",
                            MessageBoxButtons.OK,
                            MessageBoxIcon.Warning
                        );
                        return false;
                    }

                    RegistrarLog($"✅ Chrome OK: {chrome}");

                    // Verificar número
                    string numero = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "5519989102583");
                    if (string.IsNullOrEmpty(numero))
                    {
                        RegistrarLog("❌ Número não configurado");
                        numero = "5519989102583";
                    }

                    RegistrarLog($"✅ Número OK: {numero}");
                    RegistrarLog("✅ Sistema pronto para uso");

                    return true;
                }
                catch (Exception ex)
                {
                    RegistrarLog($"❌ Erro ao testar conexão: {ex.Message}");
                    return false;
                }
            });
        }

        // =====================================================
        // MÉTODO: VERIFICAR CONFIGURAÇÕES
        // =====================================================
        public static bool VerificarConfiguracoes(out string mensagemErro)
        {
            mensagemErro = "";

            try
            {
                RegistrarLog("🔍 Verificando configurações...");

                // Verificar número
                string numero = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "");
                if (string.IsNullOrEmpty(numero))
                {
                    mensagemErro = "Número do WhatsApp não configurado";
                    RegistrarLog($"❌ {mensagemErro}");
                    return false;
                }

                RegistrarLog($"✅ Número configurado: {numero}");

                // Chrome é opcional (pode usar navegador padrão)
                string chrome = EncontrarChrome();
                if (chrome == null)
                {
                    RegistrarLog("⚠️ Chrome não encontrado (tentará navegador padrão)");
                }

                return true;
            }
            catch (Exception ex)
            {
                mensagemErro = ex.Message;
                RegistrarLog($"❌ Erro ao verificar: {ex.Message}");
                return false;
            }
        }

        // =====================================================
        // MÉTODOS AUXILIARES
        // =====================================================
        
        public static bool VerificarChrome()
        {
            return EncontrarChrome() != null;
        }

        public static string ObterMensagemErroConfiguracao()
        {
            string numero = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "");
            if (string.IsNullOrEmpty(numero))
            {
                return "⚠️ Configure o número em: Menu > Configurações";
            }
            
            return "✅ Sistema OK";
        }

        public static string ObterCaminhoPerfilChrome()
        {
            string chrome = EncontrarChrome();
            return chrome ?? "Chrome não encontrado";
        }

        public static void AbrirParaObterIdGrupo()
        {
            MessageBox.Show(
                "📱 COMO CONFIGURAR O NÚMERO:\n\n" +
                "PARA CONTATOS:\n" +
                "━━━━━━━━━━━━━━━━━━\n" +
                "Use: 5519989102583\n" +
                "(código país + DDD + número)\n\n" +
                "PARA GRUPOS:\n" +
                "━━━━━━━━━━━━━━━━━━\n" +
                "1. Abra WhatsApp Web\n" +
                "2. Abra o grupo\n" +
                "3. Clique no nome\n" +
                "4. Copie o ID\n\n" +
                "💡 Número pré-configurado:\n" +
                "+55 19 98910-2583",
                "Ajuda",
                MessageBoxButtons.OK,
                MessageBoxIcon.Information
            );
        }

        public static void AbrirWhatsAppWeb()
        {
            try
            {
                RegistrarLog("🌐 Abrindo WhatsApp Web...");
                string url = "https://web.whatsapp.com";
                AbrirNoChrome(url);
            }
            catch (Exception ex)
            {
                RegistrarLog($"❌ Erro: {ex.Message}");
            }
        }

        public static async Task<string> ObterStatusInstancia()
        {
            return await Task.Run(() =>
            {
                if (VerificarChrome())
                {
                    return "✅ Chrome instalado\n✅ Sistema pronto";
                }
                else
                {
                    return "⚠️ Chrome não encontrado\n(usará navegador padrão)";
                }
            });
        }

        public static void FecharDriver() { }
        public static bool VerificarChromeDriver() { return true; }

        // =====================================================
        // MÉTODO: ABRIR PASTA DE LOGS
        // =====================================================
        public static void AbrirPastaLogs()
        {
            try
            {
                InicializarLogs();
                Process.Start("explorer.exe", pastaLogs);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao abrir pasta de logs: {ex.Message}");
            }
        }
    }
}

using System;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    /// <summary>
    /// Gerencia a sessão do usuário logado no sistema
    /// </summary>
    public static class SessaoUsuario
    {
        public static int? IdUsuario { get; private set; }
        public static string NomeUsuario { get; private set; }
        public static string LoginUsuario { get; private set; }
        public static string TipoUsuario { get; private set; } // "ADMIN" ou "USUARIO"
        public static string Telefone { get; private set; }
        public static string Email { get; private set; }
        public static byte[] FotoPerfil { get; private set; }
        public static bool EstaLogado => IdUsuario.HasValue;
        public static bool IsAdmin => TipoUsuario == "ADMIN";

        public static void Iniciar(int idUsuario, string nome, string login, string tipo, string telefone = null, string email = null, byte[] foto = null)
        {
            IdUsuario = idUsuario;
            NomeUsuario = nome;
            LoginUsuario = login;
            TipoUsuario = tipo;
            Telefone = telefone;
            Email = email;
            FotoPerfil = foto;

            RegistrarAcesso("LOGIN");
            AtualizarUltimoAcesso();
        }

        public static void Encerrar()
        {
            if (EstaLogado)
            {
                RegistrarAcesso("LOGOUT");
            }

            IdUsuario = null;
            NomeUsuario = null;
            LoginUsuario = null;
            TipoUsuario = null;
            Telefone = null;
            Email = null;
            FotoPerfil = null;
        }

        public static void AtualizarDados(string nome, string telefone, string email, byte[] foto)
        {
            NomeUsuario = nome;
            Telefone = telefone;
            Email = email;
            FotoPerfil = foto;
        }

        private static void RegistrarAcesso(string tipoAcao)
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "INSERT INTO log_acessos (id_usuario, tipo_acao) VALUES (@id, @tipo)";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@id", IdUsuario);
                    cmd.Parameters.AddWithValue("@tipo", tipoAcao);
                    cmd.ExecuteNonQuery();
                }
            }
            catch { }
        }

        private static void AtualizarUltimoAcesso()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "UPDATE usuarios SET data_ultimo_acesso = NOW() WHERE id_usuario = @id";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@id", IdUsuario);
                    cmd.ExecuteNonQuery();
                }
            }
            catch { }
        }
    }
}

using MySql.Data.MySqlClient;
using System;

namespace SistemaManutencao
{
    public static class ConexaoDB
    {
        private const string V = "Server=localhost;Database=manutencao_db;Uid=root;Pwd=;";

        // String de conexão - AJUSTE AQUI SEUS DADOS
        private static string connectionString = V;

        /// <summary>
        /// Obtém uma conexão aberta com o banco de dados
        /// </summary>
        public static MySqlConnection ObterConexao()
        {
            try
            {
                MySqlConnection conn = new MySqlConnection(connectionString);
                conn.Open();
                return conn;
            }
            catch (Exception ex)
            {
                throw new Exception("Erro ao conectar ao banco de dados: " + ex.Message);
            }
        }

        /// <summary>
        /// Obtém uma configuração do banco de dados
        /// </summary>
        public static string ObterConfiguracao(string chave, string valorPadrao = "")
        {
            try
            {
                using (MySqlConnection conn = ObterConexao())
                {
                    string query = "SELECT valor FROM configuracoes WHERE chave = @chave";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@chave", chave);

                    object resultado = cmd.ExecuteScalar();

                    if (resultado != null && resultado != DBNull.Value)
                    {
                        return resultado.ToString();
                    }

                    return valorPadrao;
                }
            }
            catch (Exception ex)
            {
                System.Diagnostics.Debug.WriteLine($"❌ Erro ao obter configuração '{chave}': {ex.Message}");
                return valorPadrao;
            }
        }

        /// <summary>
        /// Salva ou atualiza uma configuração no banco de dados
        /// </summary>
        public static void SalvarConfiguracao(string chave, string valor)
        {
            try
            {
                using (MySqlConnection conn = ObterConexao())
                {
                    // Verifica se a configuração já existe
                    string queryVerifica = "SELECT COUNT(*) FROM configuracoes WHERE chave = @chave";
                    MySqlCommand cmdVerifica = new MySqlCommand(queryVerifica, conn);
                    cmdVerifica.Parameters.AddWithValue("@chave", chave);
                    int existe = Convert.ToInt32(cmdVerifica.ExecuteScalar());

                    string query;
                    if (existe > 0)
                    {
                        // Atualizar configuração existente
                        query = "UPDATE configuracoes SET valor = @valor WHERE chave = @chave";
                    }
                    else
                    {
                        // Inserir nova configuração
                        query = "INSERT INTO configuracoes (chave, valor) VALUES (@chave, @valor)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@chave", chave);
                    cmd.Parameters.AddWithValue("@valor", valor ?? "");
                    cmd.ExecuteNonQuery();
                }
            }
            catch (Exception ex)
            {
                throw new Exception($"Erro ao salvar configuração '{chave}': {ex.Message}");
            }
        }

        /// <summary>
        /// Testa a conexão com o banco de dados
        /// </summary>
        public static bool TestarConexao(out string mensagem)
        {
            try
            {
                using (var conn = ObterConexao())
                {
                    mensagem = "Conexão bem-sucedida!";
                    return true;
                }
            }
            catch (Exception ex)
            {
                mensagem = $"Falha ao conectar: {ex.Message}";
                return false;
            }
        }
    }
}
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace ManutencaoV2._0._1
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }
    }
}

namespace ManutencaoV2._0._1
{
    partial class Form1
    {
        /// <summary>
        /// Variável de designer necessária.
        /// </summary>
        private System.ComponentModel.IContainer components = null;

        /// <summary>
        /// Limpar os recursos que estão sendo usados.
        /// </summary>
        /// <param name="disposing">true se for necessário descartar os recursos gerenciados; caso contrário, false.</param>
        protected override void Dispose(bool disposing)
        {
            if (disposing && (components != null))
            {
                components.Dispose();
            }
            base.Dispose(disposing);
        }

        #region Código gerado pelo Windows Form Designer

        /// <summary>
        /// Método necessário para suporte ao Designer - não modifique 
        /// o conteúdo deste método com o editor de código.
        /// </summary>
        private void InitializeComponent()
        {
            this.components = new System.ComponentModel.Container();
            this.AutoScaleMode = System.Windows.Forms.AutoScaleMode.Font;
            this.ClientSize = new System.Drawing.Size(800, 450);
            this.Text = "Form1";
        }

        #endregion
    }
}


using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmAjusteDatas : Form
    {
        // Controles de Filtro
        private Panel pnlFiltros;
        private Label lblTitulo;
        private ComboBox cmbPlanta, cmbMaquina, cmbPeca, cmbStatus;
        private Button btnFiltrar, btnLimpar;

        // Grid de Manutenções
        private DataGridView dgvManutencoes;

        // Painel de Edição
        private Panel pnlEdicao;
        private Label lblInfoManutencao;
        private DateTimePicker dtpNovaData;
        private TextBox txtMotivo;
        private Button btnSalvarAlteracao, btnCancelar;

        // Dados
        private int idManutencaoSelecionada = 0;
        private DateTime dataOriginal;

        public FrmAjusteDatas()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarFiltros();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Ajuste de Datas de Manutenção";
            this.Size = new Size(1400, 800);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // ============================================
            // TÍTULO
            // ============================================
            lblTitulo = new Label
            {
                Text = "📅 Ajuste de Datas de Manutenção",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // ============================================
            // PAINEL DE FILTROS
            // ============================================
            pnlFiltros = new Panel
            {
                Location = new Point(30, 80),
                Size = new Size(1320, 120),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // Linha 1 de Filtros
            Label lblPlanta = new Label
            {
                Text = "Planta:",
                Location = new Point(20, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            cmbPlanta = new ComboBox
            {
                Location = new Point(110, 20),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };

            Label lblMaquina = new Label
            {
                Text = "Máquina:",
                Location = new Point(330, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            cmbMaquina = new ComboBox
            {
                Location = new Point(420, 20),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };

            Label lblPeca = new Label
            {
                Text = "Peça:",
                Location = new Point(740, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            cmbPeca = new ComboBox
            {
                Location = new Point(830, 20),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };

            // Linha 2 de Filtros
            Label lblStatus = new Label
            {
                Text = "Status:",
                Location = new Point(20, 65),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            cmbStatus = new ComboBox
            {
                Location = new Point(110, 65),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };
            cmbStatus.Items.AddRange(new object[] { "TODOS", "PLANEJADO", "HOJE", "ATRASADO" });
            cmbStatus.SelectedIndex = 0;

            // Botões de Filtro
            btnFiltrar = new Button
            {
                Text = "🔍 Filtrar",
                Location = new Point(330, 65),
                Size = new Size(140, 35),
                BackColor = Color.FromArgb(0, 123, 255),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnFiltrar.FlatAppearance.BorderSize = 0;
            btnFiltrar.Click += BtnFiltrar_Click;

            btnLimpar = new Button
            {
                Text = "✖ Limpar",
                Location = new Point(480, 65),
                Size = new Size(140, 35),
                BackColor = Color.FromArgb(108, 117, 125),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnLimpar.FlatAppearance.BorderSize = 0;
            btnLimpar.Click += BtnLimpar_Click;

            pnlFiltros.Controls.AddRange(new Control[] {
                lblPlanta, cmbPlanta,
                lblMaquina, cmbMaquina,
                lblPeca, cmbPeca,
                lblStatus, cmbStatus,
                btnFiltrar, btnLimpar
            });

            // ============================================
            // DATAGRIDVIEW - MANUTENÇÕES
            // ============================================
            dgvManutencoes = new DataGridView
            {
                Location = new Point(30, 220),
                Size = new Size(1320, 350),
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                RowHeadersVisible = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                Font = new Font("Segoe UI", 9)
            };
            dgvManutencoes.SelectionChanged += DgvManutencoes_SelectionChanged;

            // Estilo do cabeçalho
            dgvManutencoes.EnableHeadersVisualStyles = false;
            dgvManutencoes.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(52, 73, 94);
            dgvManutencoes.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
            dgvManutencoes.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 10, FontStyle.Bold);
            dgvManutencoes.ColumnHeadersHeight = 35;

            // Estilo das linhas alternadas
            dgvManutencoes.AlternatingRowsDefaultCellStyle.BackColor = Color.FromArgb(240, 240, 240);

            // ============================================
            // PAINEL DE EDIÇÃO
            // ============================================
            pnlEdicao = new Panel
            {
                Location = new Point(30, 590),
                Size = new Size(1320, 150),
                BackColor = Color.FromArgb(255, 243, 205),
                BorderStyle = BorderStyle.FixedSingle
            };

            Label lblTituloEdicao = new Label
            {
                Text = "✏️ EDITAR DATA DE MANUTENÇÃO",
                Location = new Point(20, 15),
                Size = new Size(400, 25),
                Font = new Font("Segoe UI", 12, FontStyle.Bold),
                ForeColor = Color.FromArgb(133, 100, 4)
            };

            lblInfoManutencao = new Label
            {
                Text = "Selecione uma manutenção acima para editar",
                Location = new Point(20, 45),
                Size = new Size(1280, 20),
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.FromArgb(133, 100, 4)
            };

            Label lblNovaData = new Label
            {
                Text = "Nova Data:",
                Location = new Point(20, 75),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            dtpNovaData = new DateTimePicker
            {
                Location = new Point(130, 75),
                Size = new Size(150, 30),
                Font = new Font("Segoe UI", 10),
                Format = DateTimePickerFormat.Short
            };

            Label lblMotivo = new Label
            {
                Text = "Motivo:",
                Location = new Point(300, 75),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10),
                TextAlign = ContentAlignment.MiddleRight
            };

            txtMotivo = new TextBox
            {
                Location = new Point(390, 75),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10),
            };
            txtMotivo.ForeColor = Color.Gray;
            txtMotivo.Text = "Ex: Horas insuficientes, antecipação por parada programada...";
            txtMotivo.GotFocus += (s, e) => {
                if (txtMotivo.Text == "Ex: Horas insuficientes, antecipação por parada programada...")
                {
                    txtMotivo.Text = "";
                    txtMotivo.ForeColor = Color.Black;
                }
            };
            txtMotivo.LostFocus += (s, e) => {
                if (string.IsNullOrWhiteSpace(txtMotivo.Text))
                {
                    txtMotivo.Text = "Ex: Horas insuficientes, antecipação por parada programada...";
                    txtMotivo.ForeColor = Color.Gray;
                }
            };

            btnSalvarAlteracao = new Button
            {
                Text = "💾 Salvar e Recalcular Próximas Datas",
                Location = new Point(810, 70),
                Size = new Size(280, 40),
                BackColor = Color.FromArgb(40, 167, 69),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand,
                Enabled = false
            };
            btnSalvarAlteracao.FlatAppearance.BorderSize = 0;
            btnSalvarAlteracao.Click += BtnSalvarAlteracao_Click;

            btnCancelar = new Button
            {
                Text = "✖ Cancelar",
                Location = new Point(1100, 70),
                Size = new Size(120, 40),
                BackColor = Color.FromArgb(108, 117, 125),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnCancelar.FlatAppearance.BorderSize = 0;
            btnCancelar.Click += (s, e) => LimparSelecao();

            pnlEdicao.Controls.AddRange(new Control[] {
                lblTituloEdicao, lblInfoManutencao,
                lblNovaData, dtpNovaData,
                lblMotivo, txtMotivo,
                btnSalvarAlteracao, btnCancelar
            });

            // ============================================
            // ADICIONAR CONTROLES AO FORMULÁRIO
            // ============================================
            this.Controls.AddRange(new Control[] {
                lblTitulo,
                pnlFiltros,
                dgvManutencoes,
                pnlEdicao
            });
        }

        private void CarregarFiltros()
        {
            try
            {
                // Carregar Plantas
                cmbPlanta.Items.Clear();
                cmbPlanta.Items.Add("TODAS");
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "SELECT DISTINCT planta FROM maquinas ORDER BY planta";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataReader reader = cmd.ExecuteReader();
                    while (reader.Read())
                    {
                        cmbPlanta.Items.Add(reader["planta"].ToString());
                    }
                }
                cmbPlanta.SelectedIndex = 0;

                // Carregar Máquinas
                cmbMaquina.Items.Clear();
                cmbMaquina.Items.Add("TODAS");
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "SELECT nome_maquina FROM maquinas ORDER BY nome_maquina";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataReader reader = cmd.ExecuteReader();
                    while (reader.Read())
                    {
                        cmbMaquina.Items.Add(reader["nome_maquina"].ToString());
                    }
                }
                cmbMaquina.SelectedIndex = 0;

                // Carregar Peças
                cmbPeca.Items.Clear();
                cmbPeca.Items.Add("TODAS");
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "SELECT nome_peca FROM pecas ORDER BY nome_peca";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataReader reader = cmd.ExecuteReader();
                    while (reader.Read())
                    {
                        cmbPeca.Items.Add(reader["nome_peca"].ToString());
                    }
                }
                cmbPeca.SelectedIndex = 0;
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar filtros: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnFiltrar_Click(object sender, EventArgs e)
        {
            CarregarManutencoes();
        }

        private void BtnLimpar_Click(object sender, EventArgs e)
        {
            cmbPlanta.SelectedIndex = 0;
            cmbMaquina.SelectedIndex = 0;
            cmbPeca.SelectedIndex = 0;
            cmbStatus.SelectedIndex = 0;
            dgvManutencoes.DataSource = null;
            LimparSelecao();
        }

        private void CarregarManutencoes()
        {
            try
            {
                string query = @"
                    SELECT 
                        m.id_manutencao AS ID,
                        m.planta AS Planta,
                        maq.nome_maquina AS Máquina,
                        p.nome_peca AS Peça,
                        m.quantidade AS Qtd,
                        DATE_FORMAT(m.data_planejada, '%d/%m/%Y') AS 'Data Planejada',
                        m.data_planejada AS data_planejada_original,
                        m.situacao AS Status,
                        m.horas_previstas AS 'Horas Previstas',
                        CASE 
                            WHEN m.data_planejada = CURDATE() THEN 0
                            WHEN m.data_planejada > CURDATE() THEN DATEDIFF(m.data_planejada, CURDATE())
                            ELSE DATEDIFF(CURDATE(), m.data_planejada) * -1
                        END AS 'Dias',
                        f.nome AS Funcionário,
                        m.observacoes AS Observações
                    FROM manutencoes m
                    INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
                    INNER JOIN pecas p ON m.id_peca = p.id_peca
                    LEFT JOIN funcionarios f ON m.id_funcionario = f.id_funcionario
                    WHERE 1=1";

                // Adicionar filtros
                if (cmbPlanta.SelectedIndex > 0)
                    query += " AND m.planta = @planta";

                if (cmbMaquina.SelectedIndex > 0)
                    query += " AND maq.nome_maquina = @maquina";

                if (cmbPeca.SelectedIndex > 0)
                    query += " AND p.nome_peca = @peca";

                if (cmbStatus.SelectedIndex > 0)
                    query += " AND m.situacao = @status";

                // Ordenar
                query += " ORDER BY m.data_planejada ASC";

                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    MySqlCommand cmd = new MySqlCommand(query, conn);

                    if (cmbPlanta.SelectedIndex > 0)
                        cmd.Parameters.AddWithValue("@planta", cmbPlanta.SelectedItem.ToString());

                    if (cmbMaquina.SelectedIndex > 0)
                        cmd.Parameters.AddWithValue("@maquina", cmbMaquina.SelectedItem.ToString());

                    if (cmbPeca.SelectedIndex > 0)
                        cmd.Parameters.AddWithValue("@peca", cmbPeca.SelectedItem.ToString());

                    if (cmbStatus.SelectedIndex > 0)
                        cmd.Parameters.AddWithValue("@status", cmbStatus.SelectedItem.ToString());

                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvManutencoes.DataSource = dt;

                    // Ocultar coluna original da data
                    if (dgvManutencoes.Columns["data_planejada_original"] != null)
                        dgvManutencoes.Columns["data_planejada_original"].Visible = false;

                    // Colorir linhas baseado no status
                    foreach (DataGridViewRow row in dgvManutencoes.Rows)
                    {
                        string status = row.Cells["Status"].Value?.ToString() ?? "";

                        if (status == "HOJE")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(255, 243, 205);
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(133, 100, 4);
                        }
                        else if (status == "ATRASADO")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(248, 215, 218);
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(114, 28, 36);
                        }
                        else if (status == "PLANEJADO")
                        {
                            int dias = Convert.ToInt32(row.Cells["Dias"].Value);
                            if (dias <= 7)
                            {
                                row.DefaultCellStyle.BackColor = Color.FromArgb(209, 236, 241);
                                row.DefaultCellStyle.ForeColor = Color.FromArgb(12, 84, 96);
                            }
                        }
                    }

                    // Ajustar largura das colunas
                    if (dgvManutencoes.Columns["ID"] != null)
                        dgvManutencoes.Columns["ID"].Width = 50;
                    if (dgvManutencoes.Columns["Planta"] != null)
                        dgvManutencoes.Columns["Planta"].Width = 100;
                    if (dgvManutencoes.Columns["Dias"] != null)
                        dgvManutencoes.Columns["Dias"].Width = 60;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar manutenções: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void DgvManutencoes_SelectionChanged(object sender, EventArgs e)
        {
            if (dgvManutencoes.SelectedRows.Count > 0)
            {
                DataGridViewRow row = dgvManutencoes.SelectedRows[0];

                idManutencaoSelecionada = Convert.ToInt32(row.Cells["ID"].Value);
                string planta = row.Cells["Planta"].Value?.ToString() ?? "";
                string maquina = row.Cells["Máquina"].Value?.ToString() ?? "";
                string peca = row.Cells["Peça"].Value?.ToString() ?? "";
                string dataStr = row.Cells["Data Planejada"].Value?.ToString() ?? "";

                // Pegar a data original (não formatada)
                dataOriginal = Convert.ToDateTime(row.Cells["data_planejada_original"].Value);

                lblInfoManutencao.Text = $"ID: {idManutencaoSelecionada} | {planta} | {maquina} | {peca} | Data Atual: {dataStr}";
                dtpNovaData.Value = dataOriginal;
                txtMotivo.Text = "";

                btnSalvarAlteracao.Enabled = true;
            }
            else
            {
                LimparSelecao();
            }
        }

        private void LimparSelecao()
        {
            idManutencaoSelecionada = 0;
            lblInfoManutencao.Text = "Selecione uma manutenção acima para editar";
            txtMotivo.Text = "";
            btnSalvarAlteracao.Enabled = false;
        }

        private void BtnSalvarAlteracao_Click(object sender, EventArgs e)
        {
            if (idManutencaoSelecionada == 0)
            {
                MessageBox.Show("Selecione uma manutenção para editar!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            if (string.IsNullOrWhiteSpace(txtMotivo.Text))
            {
                MessageBox.Show("Informe o motivo da alteração!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtMotivo.Focus();
                return;
            }

            DateTime novaData = dtpNovaData.Value.Date;

            if (novaData == dataOriginal)
            {
                MessageBox.Show("A nova data é igual à data atual. Nenhuma alteração necessária.", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Information);
                return;
            }

            var result = MessageBox.Show(
                $"Confirmar alteração?\n\n" +
                $"Data Atual: {dataOriginal:dd/MM/yyyy}\n" +
                $"Nova Data: {novaData:dd/MM/yyyy}\n" +
                $"Diferença: {(novaData - dataOriginal).Days} dias\n\n" +
                $"Motivo: {txtMotivo.Text}\n\n" +
                $"⚠️ Isso irá recalcular TODAS as próximas manutenções desta máquina + peça!",
                "Confirmar Alteração",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Question
            );

            if (result == DialogResult.Yes)
            {
                RecalcularDatasManutencao(novaData);
            }
        }

        private void RecalcularDatasManutencao(DateTime novaData)
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // 1. Obter informações da manutenção atual
                    string queryInfo = @"
                        SELECT m.id_maquina, m.id_peca, m.data_planejada, m.planta
                        FROM manutencoes m
                        WHERE m.id_manutencao = @id";

                    MySqlCommand cmdInfo = new MySqlCommand(queryInfo, conn);
                    cmdInfo.Parameters.AddWithValue("@id", idManutencaoSelecionada);
                    MySqlDataReader reader = cmdInfo.ExecuteReader();

                    if (!reader.Read())
                    {
                        reader.Close();
                        MessageBox.Show("Manutenção não encontrada!", "Erro",
                            MessageBoxButtons.OK, MessageBoxIcon.Error);
                        return;
                    }

                    int idMaquina = Convert.ToInt32(reader["id_maquina"]);
                    int idPeca = Convert.ToInt32(reader["id_peca"]);
                    DateTime dataAtual = Convert.ToDateTime(reader["data_planejada"]);
                    string planta = reader["planta"].ToString();
                    reader.Close();

                    // 2. Calcular diferença em dias (intervalo)
                    int diferencaDias = (int)(novaData - dataAtual).TotalDays;

                    // 3. Buscar todas as próximas manutenções da mesma máquina + peça
                    string queryProximas = @"
                        SELECT id_manutencao, data_planejada, situacao
                        FROM manutencoes
                        WHERE id_maquina = @id_maquina
                        AND id_peca = @id_peca
                        AND planta = @planta
                        AND data_planejada >= @data_atual
                        AND situacao IN ('PLANEJADO', 'HOJE', 'ATRASADO')
                        ORDER BY data_planejada";

                    MySqlCommand cmdProximas = new MySqlCommand(queryProximas, conn);
                    cmdProximas.Parameters.AddWithValue("@id_maquina", idMaquina);
                    cmdProximas.Parameters.AddWithValue("@id_peca", idPeca);
                    cmdProximas.Parameters.AddWithValue("@planta", planta);
                    cmdProximas.Parameters.AddWithValue("@data_atual", dataAtual);

                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmdProximas);
                    DataTable dtProximas = new DataTable();
                    adapter.Fill(dtProximas);

                    // 4. Atualizar cada manutenção
                    int totalAtualizadas = 0;
                    foreach (DataRow row in dtProximas.Rows)
                    {
                        int idManut = Convert.ToInt32(row["id_manutencao"]);
                        DateTime dataManut = Convert.ToDateTime(row["data_planejada"]);

                        // Calcular nova data adicionando a diferença
                        DateTime novaDataManut = dataManut.AddDays(diferencaDias);

                        // Determinar novo status
                        string novoStatus = "PLANEJADO";
                        if (novaDataManut.Date == DateTime.Today)
                            novoStatus = "HOJE";
                        else if (novaDataManut.Date < DateTime.Today)
                            novoStatus = "ATRASADO";

                        // Atualizar no banco
                        string queryUpdate = @"
                            UPDATE manutencoes 
                            SET data_planejada = @nova_data,
                                situacao = @novo_status,
                                observacoes = CONCAT(IFNULL(observacoes, ''), 
                                    CHAR(10), 
                                    'AJUSTE: ', DATE_FORMAT(NOW(), '%d/%m/%Y %H:%i'), 
                                    ' - Data alterada de ', DATE_FORMAT(@data_antiga, '%d/%m/%Y'),
                                    ' para ', DATE_FORMAT(@nova_data, '%d/%m/%Y'),
                                    ' - Motivo: ', @motivo)
                            WHERE id_manutencao = @id";

                        MySqlCommand cmdUpdate = new MySqlCommand(queryUpdate, conn);
                        cmdUpdate.Parameters.AddWithValue("@nova_data", novaDataManut);
                        cmdUpdate.Parameters.AddWithValue("@novo_status", novoStatus);
                        cmdUpdate.Parameters.AddWithValue("@data_antiga", dataManut);
                        cmdUpdate.Parameters.AddWithValue("@motivo", txtMotivo.Text);
                        cmdUpdate.Parameters.AddWithValue("@id", idManut);
                        cmdUpdate.ExecuteNonQuery();

                        totalAtualizadas++;
                    }

                    MessageBox.Show(
                        $"✅ Datas recalculadas com sucesso!\n\n" +
                        $"Total de manutenções atualizadas: {totalAtualizadas}\n" +
                        $"Diferença aplicada: {diferencaDias} dias",
                        "Sucesso",
                        MessageBoxButtons.OK,
                        MessageBoxIcon.Information
                    );

                    // Recarregar grid
                    CarregarManutencoes();
                    LimparSelecao();
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao recalcular datas: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}
using System;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmAlterarSenha : Form
    {
        private TextBox txtSenhaAtual, txtNovaSenha, txtConfirmarSenha;
        private Button btnSalvar, btnCancelar;

        public FrmAlterarSenha()
        {
            ConfigurarFormulario();
            CriarInterface();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Alterar Senha";
            this.Size = new Size(450, 350);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterParent;
            this.FormBorderStyle = FormBorderStyle.FixedDialog;
            this.MaximizeBox = false;
            this.MinimizeBox = false;
        }

        private void CriarInterface()
        {
            Label lblTitulo = new Label
            {
                Text = "🔒 Alterar Senha",
                Font = new Font("Segoe UI", 16, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            int yPos = 80;

            // Senha Atual
            Label lblSenhaAtual = new Label
            {
                Text = "Senha Atual:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtSenhaAtual = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(220, 30),
                Font = new Font("Segoe UI", 10),
                PasswordChar = '●'
            };
            yPos += 50;

            // Nova Senha
            Label lblNovaSenha = new Label
            {
                Text = "Nova Senha:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtNovaSenha = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(220, 30),
                Font = new Font("Segoe UI", 10),
                PasswordChar = '●'
            };
            yPos += 50;

            // Confirmar Senha
            Label lblConfirmarSenha = new Label
            {
                Text = "Confirmar Senha:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtConfirmarSenha = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(220, 30),
                Font = new Font("Segoe UI", 10),
                PasswordChar = '●'
            };

            // Botões
            btnSalvar = new Button
            {
                Text = "Salvar",
                Location = new Point(180, 250),
                Size = new Size(100, 35),
                BackColor = Color.FromArgb(46, 204, 113),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnSalvar.FlatAppearance.BorderSize = 0;
            btnSalvar.Click += BtnSalvar_Click;

            btnCancelar = new Button
            {
                Text = "Cancelar",
                Location = new Point(290, 250),
                Size = new Size(100, 35),
                BackColor = Color.FromArgb(149, 165, 166),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnCancelar.FlatAppearance.BorderSize = 0;
            btnCancelar.Click += (s, e) => this.Close();

            this.Controls.AddRange(new Control[] {
                lblTitulo,
                lblSenhaAtual, txtSenhaAtual,
                lblNovaSenha, txtNovaSenha,
                lblConfirmarSenha, txtConfirmarSenha,
                btnSalvar, btnCancelar
            });
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            if (string.IsNullOrEmpty(txtSenhaAtual.Text) || 
                string.IsNullOrEmpty(txtNovaSenha.Text) || 
                string.IsNullOrEmpty(txtConfirmarSenha.Text))
            {
                MessageBox.Show("Preencha todos os campos!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            if (txtNovaSenha.Text != txtConfirmarSenha.Text)
            {
                MessageBox.Show("A nova senha e a confirmação não coincidem!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            if (txtNovaSenha.Text.Length < 6)
            {
                MessageBox.Show("A senha deve ter no mínimo 6 caracteres!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // Verificar senha atual
                    string queryVerificar = "SELECT senha FROM usuarios WHERE id_usuario = @id";
                    MySqlCommand cmdVerificar = new MySqlCommand(queryVerificar, conn);
                    cmdVerificar.Parameters.AddWithValue("@id", SessaoUsuario.IdUsuario);

                    string senhaAtual = cmdVerificar.ExecuteScalar()?.ToString();

                    if (senhaAtual != txtSenhaAtual.Text)
                    {
                        MessageBox.Show("Senha atual incorreta!", "Erro",
                            MessageBoxButtons.OK, MessageBoxIcon.Error);
                        return;
                    }

                    // Atualizar senha
                    string queryAtualizar = "UPDATE usuarios SET senha = @senha WHERE id_usuario = @id";
                    MySqlCommand cmdAtualizar = new MySqlCommand(queryAtualizar, conn);
                    cmdAtualizar.Parameters.AddWithValue("@senha", txtNovaSenha.Text);
                    cmdAtualizar.Parameters.AddWithValue("@id", SessaoUsuario.IdUsuario);

                    cmdAtualizar.ExecuteNonQuery();

                    MessageBox.Show("Senha alterada com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    this.Close();
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao alterar senha: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}

using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmCadastroGeral : Form
    {
        // Controles de Formulário
        private Label lblTitulo;
        private TextBox txtIdMaquina, txtMaquina, txtIdPeca, txtPeca, txtIdFuncionario, txtFuncionario;
        private TextBox txtPlanta, txtCodigoPeca, txtTelefone, txtWhatsApp, txtCargo;
        private TextBox txtStatusMaquina, txtStatusFuncionario;
        private TabControl tabControl;
        private TabPage tabMaquinas, tabPecas, tabFuncionarios;
        
        // Grids
        private DataGridView dgvMaquinas, dgvPecas, dgvFuncionarios;
        
        // Controle - Máquinas
        private int idMaquinaSelecionada = 0;
        private bool modoEdicaoMaquina = false;
        
        // Controle - Peças
        private int idPecaSelecionada = 0;
        private bool modoEdicaoPeca = false;
        
        // Controle - Funcionários
        private int idFuncionarioSelecionado = 0;
        private bool modoEdicaoFuncionario = false;

        public FrmCadastroGeral()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarDados();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Cadastro Geral";
            this.Size = new Size(1250, 750);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // Título
            lblTitulo = new Label
            {
                Text = "📋 Cadastro Geral",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // TabControl
            tabControl = new TabControl
            {
                Location = new Point(30, 80),
                Size = new Size(1170, 620),
                Font = new Font("Segoe UI", 10)
            };

            // Criar abas
            CriarAbaMaquinas();
            CriarAbaPecas();
            CriarAbaFuncionarios();

            // Adicionar controles ao formulário
            this.Controls.AddRange(new Control[] { lblTitulo, tabControl });
        }

        #region ABA MÁQUINAS
        
        private void CriarAbaMaquinas()
        {
            tabMaquinas = new TabPage("⚙️ Máquinas");
            
            // Painel de Cadastro
            Panel pnlMaquinas = new Panel
            {
                Location = new Point(10, 10),
                Size = new Size(1135, 200),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // ID
            Label lblIdMaquina = new Label
            {
                Text = "ID:",
                Location = new Point(20, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtIdMaquina = new TextBox
            {
                Location = new Point(130, 20),
                Size = new Size(100, 30),
                Enabled = false,
                BackColor = Color.LightGray,
                Font = new Font("Segoe UI", 10)
            };

            // Planta (CAMPO LIVRE)
            Label lblPlantaMaquina = new Label
            {
                Text = "Planta:*",
                Location = new Point(20, 60),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtPlanta = new TextBox
            {
                Location = new Point(130, 60),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Nome da Máquina
            Label lblNomeMaquina = new Label
            {
                Text = "Nome Máquina:*",
                Location = new Point(350, 60),
                Size = new Size(130, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtMaquina = new TextBox
            {
                Location = new Point(490, 60),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Status
            Label lblStatusMaquina = new Label
            {
                Text = "Status:",
                Location = new Point(20, 100),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtStatusMaquina = new TextBox
            {
                Location = new Point(130, 100),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Botões Máquinas
            int btnY = 145;
            int btnX = 20;

            Button btnNovoMaquina = CriarBotao("➕ Novo", btnX, btnY, Color.FromArgb(0, 123, 255));
            btnNovoMaquina.Click += BtnNovoMaquina_Click;
            btnX += 120;

            Button btnSalvarMaquina = CriarBotao("💾 Salvar", btnX, btnY, Color.FromArgb(40, 167, 69));
            btnSalvarMaquina.Click += BtnSalvarMaquina_Click;
            btnX += 120;

            Button btnEditarMaquina = CriarBotao("✏️ Editar", btnX, btnY, Color.FromArgb(255, 193, 7));
            btnEditarMaquina.Click += BtnEditarMaquina_Click;
            btnX += 120;

            Button btnExcluirMaquina = CriarBotao("🗑️ Excluir", btnX, btnY, Color.FromArgb(220, 53, 69));
            btnExcluirMaquina.Click += BtnExcluirMaquina_Click;
            btnX += 120;

            Button btnCancelarMaquina = CriarBotao("✖ Cancelar", btnX, btnY, Color.FromArgb(108, 117, 125));
            btnCancelarMaquina.Click += BtnCancelarMaquina_Click;

            pnlMaquinas.Controls.AddRange(new Control[] {
                lblIdMaquina, txtIdMaquina,
                lblPlantaMaquina, txtPlanta,
                lblNomeMaquina, txtMaquina,
                lblStatusMaquina, txtStatusMaquina,
                btnNovoMaquina, btnSalvarMaquina, btnEditarMaquina, btnExcluirMaquina, btnCancelarMaquina
            });

            // DataGridView Máquinas
            dgvMaquinas = CriarDataGridView();
            dgvMaquinas.Location = new Point(10, 220);
            dgvMaquinas.Size = new Size(1135, 350);
            dgvMaquinas.SelectionChanged += (s, e) => HabilitarBotoesGrid(dgvMaquinas, btnEditarMaquina, btnExcluirMaquina);

            tabMaquinas.Controls.AddRange(new Control[] { pnlMaquinas, dgvMaquinas });
            tabControl.TabPages.Add(tabMaquinas);
        }

        #endregion

        #region ABA PEÇAS

        private void CriarAbaPecas()
        {
            tabPecas = new TabPage("🔧 Peças");
            
            // Painel de Cadastro
            Panel pnlPecas = new Panel
            {
                Location = new Point(10, 10),
                Size = new Size(1135, 200),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // ID
            Label lblIdPeca = new Label
            {
                Text = "ID:",
                Location = new Point(20, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtIdPeca = new TextBox
            {
                Location = new Point(130, 20),
                Size = new Size(100, 30),
                Enabled = false,
                BackColor = Color.LightGray,
                Font = new Font("Segoe UI", 10)
            };

            // Nome da Peça
            Label lblNomePeca = new Label
            {
                Text = "Nome Peça:*",
                Location = new Point(20, 60),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtPeca = new TextBox
            {
                Location = new Point(130, 60),
                Size = new Size(500, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Código da Peça
            Label lblCodigoPeca = new Label
            {
                Text = "Código:",
                Location = new Point(650, 60),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtCodigoPeca = new TextBox
            {
                Location = new Point(740, 60),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Botões Peças
            int btnY = 105;
            int btnX = 20;

            Button btnNovoPeca = CriarBotao("➕ Novo", btnX, btnY, Color.FromArgb(0, 123, 255));
            btnNovoPeca.Click += BtnNovoPeca_Click;
            btnX += 120;

            Button btnSalvarPeca = CriarBotao("💾 Salvar", btnX, btnY, Color.FromArgb(40, 167, 69));
            btnSalvarPeca.Click += BtnSalvarPeca_Click;
            btnX += 120;

            Button btnEditarPeca = CriarBotao("✏️ Editar", btnX, btnY, Color.FromArgb(255, 193, 7));
            btnEditarPeca.Click += BtnEditarPeca_Click;
            btnX += 120;

            Button btnExcluirPeca = CriarBotao("🗑️ Excluir", btnX, btnY, Color.FromArgb(220, 53, 69));
            btnExcluirPeca.Click += BtnExcluirPeca_Click;
            btnX += 120;

            Button btnCancelarPeca = CriarBotao("✖ Cancelar", btnX, btnY, Color.FromArgb(108, 117, 125));
            btnCancelarPeca.Click += BtnCancelarPeca_Click;

            pnlPecas.Controls.AddRange(new Control[] {
                lblIdPeca, txtIdPeca,
                lblNomePeca, txtPeca,
                lblCodigoPeca, txtCodigoPeca,
                btnNovoPeca, btnSalvarPeca, btnEditarPeca, btnExcluirPeca, btnCancelarPeca
            });

            // DataGridView Peças
            dgvPecas = CriarDataGridView();
            dgvPecas.Location = new Point(10, 220);
            dgvPecas.Size = new Size(1135, 350);
            dgvPecas.SelectionChanged += (s, e) => HabilitarBotoesGrid(dgvPecas, btnEditarPeca, btnExcluirPeca);

            tabPecas.Controls.AddRange(new Control[] { pnlPecas, dgvPecas });
            tabControl.TabPages.Add(tabPecas);
        }

        #endregion

        #region ABA FUNCIONÁRIOS

        private void CriarAbaFuncionarios()
        {
            tabFuncionarios = new TabPage("👤 Funcionários");
            
            // Painel de Cadastro
            Panel pnlFuncionarios = new Panel
            {
                Location = new Point(10, 10),
                Size = new Size(1135, 200),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // ID
            Label lblIdFuncionario = new Label
            {
                Text = "ID:",
                Location = new Point(20, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtIdFuncionario = new TextBox
            {
                Location = new Point(130, 20),
                Size = new Size(100, 30),
                Enabled = false,
                BackColor = Color.LightGray,
                Font = new Font("Segoe UI", 10)
            };

            // Nome
            Label lblNomeFuncionario = new Label
            {
                Text = "Nome:*",
                Location = new Point(20, 60),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtFuncionario = new TextBox
            {
                Location = new Point(130, 60),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Telefone
            Label lblTelefone = new Label
            {
                Text = "Telefone:",
                Location = new Point(450, 60),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtTelefone = new TextBox
            {
                Location = new Point(540, 60),
                Size = new Size(180, 30),
                Font = new Font("Segoe UI", 10)
            };

            // WhatsApp
            Label lblWhatsApp = new Label
            {
                Text = "WhatsApp:",
                Location = new Point(740, 60),
                Size = new Size(90, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtWhatsApp = new TextBox
            {
                Location = new Point(840, 60),
                Size = new Size(180, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Cargo
            Label lblCargo = new Label
            {
                Text = "Cargo:",
                Location = new Point(20, 100),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtCargo = new TextBox
            {
                Location = new Point(130, 100),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Status
            Label lblStatusFuncionario = new Label
            {
                Text = "Status:",
                Location = new Point(450, 100),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtStatusFuncionario = new TextBox
            {
                Location = new Point(540, 100),
                Size = new Size(150, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Botões Funcionários
            int btnY = 145;
            int btnX = 20;

            Button btnNovoFuncionario = CriarBotao("➕ Novo", btnX, btnY, Color.FromArgb(0, 123, 255));
            btnNovoFuncionario.Click += BtnNovoFuncionario_Click;
            btnX += 120;

            Button btnSalvarFuncionario = CriarBotao("💾 Salvar", btnX, btnY, Color.FromArgb(40, 167, 69));
            btnSalvarFuncionario.Click += BtnSalvarFuncionario_Click;
            btnX += 120;

            Button btnEditarFuncionario = CriarBotao("✏️ Editar", btnX, btnY, Color.FromArgb(255, 193, 7));
            btnEditarFuncionario.Click += BtnEditarFuncionario_Click;
            btnX += 120;

            Button btnExcluirFuncionario = CriarBotao("🗑️ Excluir", btnX, btnY, Color.FromArgb(220, 53, 69));
            btnExcluirFuncionario.Click += BtnExcluirFuncionario_Click;
            btnX += 120;

            Button btnCancelarFuncionario = CriarBotao("✖ Cancelar", btnX, btnY, Color.FromArgb(108, 117, 125));
            btnCancelarFuncionario.Click += BtnCancelarFuncionario_Click;

            pnlFuncionarios.Controls.AddRange(new Control[] {
                lblIdFuncionario, txtIdFuncionario,
                lblNomeFuncionario, txtFuncionario,
                lblTelefone, txtTelefone,
                lblWhatsApp, txtWhatsApp,
                lblCargo, txtCargo,
                lblStatusFuncionario, txtStatusFuncionario,
                btnNovoFuncionario, btnSalvarFuncionario, btnEditarFuncionario, 
                btnExcluirFuncionario, btnCancelarFuncionario
            });

            // DataGridView Funcionários
            dgvFuncionarios = CriarDataGridView();
            dgvFuncionarios.Location = new Point(10, 220);
            dgvFuncionarios.Size = new Size(1135, 350);
            dgvFuncionarios.SelectionChanged += (s, e) => HabilitarBotoesGrid(dgvFuncionarios, btnEditarFuncionario, btnExcluirFuncionario);

            tabFuncionarios.Controls.AddRange(new Control[] { pnlFuncionarios, dgvFuncionarios });
            tabControl.TabPages.Add(tabFuncionarios);
        }

        #endregion

        #region MÉTODOS AUXILIARES

        private Button CriarBotao(string texto, int x, int y, Color cor)
        {
            Button btn = new Button
            {
                Text = texto,
                Location = new Point(x, y),
                Size = new Size(110, 40),
                BackColor = cor,
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btn.FlatAppearance.BorderSize = 0;
            return btn;
        }

        private DataGridView CriarDataGridView()
        {
            DataGridView dgv = new DataGridView
            {
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                RowHeadersVisible = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                Font = new Font("Segoe UI", 9)
            };

            dgv.EnableHeadersVisualStyles = false;
            dgv.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(52, 73, 94);
            dgv.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
            dgv.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 10, FontStyle.Bold);
            dgv.ColumnHeadersHeight = 35;

            return dgv;
        }

        private void HabilitarBotoesGrid(DataGridView dgv, Button btnEditar, Button btnExcluir)
        {
            if (dgv.SelectedRows.Count > 0)
            {
                btnEditar.Enabled = true;
                btnExcluir.Enabled = true;
            }
            else
            {
                btnEditar.Enabled = false;
                btnExcluir.Enabled = false;
            }
        }

        #endregion

        #region CARREGAMENTO DE DADOS

        private void CarregarDados()
        {
            CarregarMaquinas();
            CarregarPecas();
            CarregarFuncionarios();
        }

        private void CarregarMaquinas()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_maquina AS ID,
                            planta AS Planta,
                            nome_maquina AS 'Nome Máquina',
                            status_operacional AS Status,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y') AS 'Data Cadastro'
                        FROM maquinas
                        ORDER BY id_maquina DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvMaquinas.DataSource = dt;

                    if (dgvMaquinas.Columns["ID"] != null)
                        dgvMaquinas.Columns["ID"].Width = 60;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar máquinas: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CarregarPecas()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_peca AS ID,
                            nome_peca AS 'Nome Peça',
                            codigo_peca AS Código,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y') AS 'Data Cadastro'
                        FROM pecas
                        ORDER BY id_peca DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvPecas.DataSource = dt;

                    if (dgvPecas.Columns["ID"] != null)
                        dgvPecas.Columns["ID"].Width = 60;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar peças: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CarregarFuncionarios()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_funcionario AS ID,
                            nome AS Nome,
                            telefone AS Telefone,
                            whatsapp AS WhatsApp,
                            cargo AS Cargo,
                            status AS Status,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y') AS 'Data Cadastro'
                        FROM funcionarios
                        ORDER BY id_funcionario DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvFuncionarios.DataSource = dt;

                    if (dgvFuncionarios.Columns["ID"] != null)
                        dgvFuncionarios.Columns["ID"].Width = 60;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar funcionários: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        #endregion

        #region MÁQUINAS - EVENTOS

        private void BtnNovoMaquina_Click(object sender, EventArgs e)
        {
            txtIdMaquina.Clear();
            txtPlanta.Clear();
            txtMaquina.Clear();
            txtStatusMaquina.Clear();
            
            txtPlanta.Enabled = true;
            txtMaquina.Enabled = true;
            txtStatusMaquina.Enabled = true;
            
            modoEdicaoMaquina = false;
            idMaquinaSelecionada = 0;
            txtPlanta.Focus();
        }

        private void BtnSalvarMaquina_Click(object sender, EventArgs e)
        {
            if (string.IsNullOrWhiteSpace(txtPlanta.Text))
            {
                MessageBox.Show("Informe a planta!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtPlanta.Focus();
                return;
            }

            if (string.IsNullOrWhiteSpace(txtMaquina.Text))
            {
                MessageBox.Show("Informe o nome da máquina!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtMaquina.Focus();
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query;

                    if (modoEdicaoMaquina && idMaquinaSelecionada > 0)
                    {
                        query = @"UPDATE maquinas 
                                SET planta = @planta,
                                    nome_maquina = @nome,
                                    status_operacional = @status
                                WHERE id_maquina = @id";
                    }
                    else
                    {
                        query = @"INSERT INTO maquinas 
                                (planta, nome_maquina, status_operacional)
                                VALUES (@planta, @nome, @status)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@planta", txtPlanta.Text.Trim());
                    cmd.Parameters.AddWithValue("@nome", txtMaquina.Text.Trim());
                    cmd.Parameters.AddWithValue("@status", string.IsNullOrWhiteSpace(txtStatusMaquina.Text) ? "ativo" : txtStatusMaquina.Text.Trim());

                    if (modoEdicaoMaquina && idMaquinaSelecionada > 0)
                        cmd.Parameters.AddWithValue("@id", idMaquinaSelecionada);

                    cmd.ExecuteNonQuery();

                    MessageBox.Show("Máquina salva com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarMaquinas();
                    BtnCancelarMaquina_Click(sender, e);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnEditarMaquina_Click(object sender, EventArgs e)
        {
            if (dgvMaquinas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma máquina para editar!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            DataGridViewRow row = dgvMaquinas.SelectedRows[0];
            idMaquinaSelecionada = Convert.ToInt32(row.Cells["ID"].Value);
            
            txtIdMaquina.Text = idMaquinaSelecionada.ToString();
            txtPlanta.Text = row.Cells["Planta"].Value.ToString();
            txtMaquina.Text = row.Cells["Nome Máquina"].Value.ToString();
            txtStatusMaquina.Text = row.Cells["Status"].Value?.ToString() ?? "";

            txtPlanta.Enabled = true;
            txtMaquina.Enabled = true;
            txtStatusMaquina.Enabled = true;
            
            modoEdicaoMaquina = true;
            txtPlanta.Focus();
        }

        private void BtnExcluirMaquina_Click(object sender, EventArgs e)
        {
            if (dgvMaquinas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma máquina para excluir!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            var result = MessageBox.Show(
                "Deseja realmente excluir esta máquina?\n\nAtenção: Isso pode afetar manutenções vinculadas!",
                "Confirmar Exclusão",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                try
                {
                    int id = Convert.ToInt32(dgvMaquinas.SelectedRows[0].Cells["ID"].Value);

                    using (MySqlConnection conn = ConexaoDB.ObterConexao())
                    {
                        string query = "DELETE FROM maquinas WHERE id_maquina = @id";
                        MySqlCommand cmd = new MySqlCommand(query, conn);
                        cmd.Parameters.AddWithValue("@id", id);
                        cmd.ExecuteNonQuery();
                    }

                    MessageBox.Show("Máquina excluída com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarMaquinas();
                    BtnCancelarMaquina_Click(sender, e);
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao excluir: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void BtnCancelarMaquina_Click(object sender, EventArgs e)
        {
            txtIdMaquina.Clear();
            txtPlanta.Clear();
            txtMaquina.Clear();
            txtStatusMaquina.Clear();
            
            txtPlanta.Enabled = false;
            txtMaquina.Enabled = false;
            txtStatusMaquina.Enabled = false;
            
            idMaquinaSelecionada = 0;
            modoEdicaoMaquina = false;
        }

        #endregion

        #region PEÇAS - EVENTOS

        private void BtnNovoPeca_Click(object sender, EventArgs e)
        {
            txtIdPeca.Clear();
            txtPeca.Clear();
            txtCodigoPeca.Clear();
            
            txtPeca.Enabled = true;
            txtCodigoPeca.Enabled = true;
            
            modoEdicaoPeca = false;
            idPecaSelecionada = 0;
            txtPeca.Focus();
        }

        private void BtnSalvarPeca_Click(object sender, EventArgs e)
        {
            if (string.IsNullOrWhiteSpace(txtPeca.Text))
            {
                MessageBox.Show("Informe o nome da peça!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtPeca.Focus();
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query;

                    if (modoEdicaoPeca && idPecaSelecionada > 0)
                    {
                        query = @"UPDATE pecas 
                                SET nome_peca = @nome,
                                    codigo_peca = @codigo
                                WHERE id_peca = @id";
                    }
                    else
                    {
                        query = @"INSERT INTO pecas (nome_peca, codigo_peca)
                                VALUES (@nome, @codigo)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@nome", txtPeca.Text.Trim());
                    cmd.Parameters.AddWithValue("@codigo", txtCodigoPeca.Text.Trim());

                    if (modoEdicaoPeca && idPecaSelecionada > 0)
                        cmd.Parameters.AddWithValue("@id", idPecaSelecionada);

                    cmd.ExecuteNonQuery();

                    MessageBox.Show("Peça salva com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarPecas();
                    BtnCancelarPeca_Click(sender, e);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnEditarPeca_Click(object sender, EventArgs e)
        {
            if (dgvPecas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma peça para editar!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            DataGridViewRow row = dgvPecas.SelectedRows[0];
            idPecaSelecionada = Convert.ToInt32(row.Cells["ID"].Value);
            
            txtIdPeca.Text = idPecaSelecionada.ToString();
            txtPeca.Text = row.Cells["Nome Peça"].Value.ToString();
            txtCodigoPeca.Text = row.Cells["Código"].Value?.ToString() ?? "";

            txtPeca.Enabled = true;
            txtCodigoPeca.Enabled = true;
            
            modoEdicaoPeca = true;
            txtPeca.Focus();
        }

        private void BtnExcluirPeca_Click(object sender, EventArgs e)
        {
            if (dgvPecas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma peça para excluir!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            var result = MessageBox.Show(
                "Deseja realmente excluir esta peça?\n\nAtenção: Isso pode afetar manutenções vinculadas!",
                "Confirmar Exclusão",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                try
                {
                    int id = Convert.ToInt32(dgvPecas.SelectedRows[0].Cells["ID"].Value);

                    using (MySqlConnection conn = ConexaoDB.ObterConexao())
                    {
                        string query = "DELETE FROM pecas WHERE id_peca = @id";
                        MySqlCommand cmd = new MySqlCommand(query, conn);
                        cmd.Parameters.AddWithValue("@id", id);
                        cmd.ExecuteNonQuery();
                    }

                    MessageBox.Show("Peça excluída com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarPecas();
                    BtnCancelarPeca_Click(sender, e);
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao excluir: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void BtnCancelarPeca_Click(object sender, EventArgs e)
        {
            txtIdPeca.Clear();
            txtPeca.Clear();
            txtCodigoPeca.Clear();
            
            txtPeca.Enabled = false;
            txtCodigoPeca.Enabled = false;
            
            idPecaSelecionada = 0;
            modoEdicaoPeca = false;
        }

        #endregion

        #region FUNCIONÁRIOS - EVENTOS

        private void BtnNovoFuncionario_Click(object sender, EventArgs e)
        {
            txtIdFuncionario.Clear();
            txtFuncionario.Clear();
            txtTelefone.Clear();
            txtWhatsApp.Clear();
            txtCargo.Clear();
            txtStatusFuncionario.Clear();
            
            txtFuncionario.Enabled = true;
            txtTelefone.Enabled = true;
            txtWhatsApp.Enabled = true;
            txtCargo.Enabled = true;
            txtStatusFuncionario.Enabled = true;
            
            modoEdicaoFuncionario = false;
            idFuncionarioSelecionado = 0;
            txtFuncionario.Focus();
        }

        private void BtnSalvarFuncionario_Click(object sender, EventArgs e)
        {
            if (string.IsNullOrWhiteSpace(txtFuncionario.Text))
            {
                MessageBox.Show("Informe o nome do funcionário!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtFuncionario.Focus();
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query;

                    if (modoEdicaoFuncionario && idFuncionarioSelecionado > 0)
                    {
                        query = @"UPDATE funcionarios 
                                SET nome = @nome,
                                    telefone = @telefone,
                                    whatsapp = @whatsapp,
                                    cargo = @cargo,
                                    status = @status
                                WHERE id_funcionario = @id";
                    }
                    else
                    {
                        query = @"INSERT INTO funcionarios 
                                (nome, telefone, whatsapp, cargo, status)
                                VALUES (@nome, @telefone, @whatsapp, @cargo, @status)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@nome", txtFuncionario.Text.Trim());
                    cmd.Parameters.AddWithValue("@telefone", txtTelefone.Text.Trim());
                    cmd.Parameters.AddWithValue("@whatsapp", txtWhatsApp.Text.Trim());
                    cmd.Parameters.AddWithValue("@cargo", txtCargo.Text.Trim());
                    cmd.Parameters.AddWithValue("@status", string.IsNullOrWhiteSpace(txtStatusFuncionario.Text) ? "ativo" : txtStatusFuncionario.Text.Trim());

                    if (modoEdicaoFuncionario && idFuncionarioSelecionado > 0)
                        cmd.Parameters.AddWithValue("@id", idFuncionarioSelecionado);

                    cmd.ExecuteNonQuery();

                    MessageBox.Show("Funcionário salvo com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarFuncionarios();
                    BtnCancelarFuncionario_Click(sender, e);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnEditarFuncionario_Click(object sender, EventArgs e)
        {
            if (dgvFuncionarios.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione um funcionário para editar!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            DataGridViewRow row = dgvFuncionarios.SelectedRows[0];
            idFuncionarioSelecionado = Convert.ToInt32(row.Cells["ID"].Value);
            
            txtIdFuncionario.Text = idFuncionarioSelecionado.ToString();
            txtFuncionario.Text = row.Cells["Nome"].Value.ToString();
            txtTelefone.Text = row.Cells["Telefone"].Value?.ToString() ?? "";
            txtWhatsApp.Text = row.Cells["WhatsApp"].Value?.ToString() ?? "";
            txtCargo.Text = row.Cells["Cargo"].Value?.ToString() ?? "";
            txtStatusFuncionario.Text = row.Cells["Status"].Value?.ToString() ?? "";

            txtFuncionario.Enabled = true;
            txtTelefone.Enabled = true;
            txtWhatsApp.Enabled = true;
            txtCargo.Enabled = true;
            txtStatusFuncionario.Enabled = true;
            
            modoEdicaoFuncionario = true;
            txtFuncionario.Focus();
        }

        private void BtnExcluirFuncionario_Click(object sender, EventArgs e)
        {
            if (dgvFuncionarios.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione um funcionário para excluir!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            var result = MessageBox.Show(
                "Deseja realmente excluir este funcionário?\n\nAtenção: Isso pode afetar manutenções vinculadas!",
                "Confirmar Exclusão",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                try
                {
                    int id = Convert.ToInt32(dgvFuncionarios.SelectedRows[0].Cells["ID"].Value);

                    using (MySqlConnection conn = ConexaoDB.ObterConexao())
                    {
                        string query = "DELETE FROM funcionarios WHERE id_funcionario = @id";
                        MySqlCommand cmd = new MySqlCommand(query, conn);
                        cmd.Parameters.AddWithValue("@id", id);
                        cmd.ExecuteNonQuery();
                    }

                    MessageBox.Show("Funcionário excluído com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarFuncionarios();
                    BtnCancelarFuncionario_Click(sender, e);
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao excluir: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void BtnCancelarFuncionario_Click(object sender, EventArgs e)
        {
            txtIdFuncionario.Clear();
            txtFuncionario.Clear();
            txtTelefone.Clear();
            txtWhatsApp.Clear();
            txtCargo.Clear();
            txtStatusFuncionario.Clear();
            
            txtFuncionario.Enabled = false;
            txtTelefone.Enabled = false;
            txtWhatsApp.Enabled = false;
            txtCargo.Enabled = false;
            txtStatusFuncionario.Enabled = false;
            
            idFuncionarioSelecionado = 0;
            modoEdicaoFuncionario = false;
        }

        #endregion
    }
}

using MySql.Data.MySqlClient;
using Mysqlx.Crud;
using System;
using System.Data;
using System.Drawing;
using System.Numerics;
using System.Windows.Forms;

namespace SistemaManutencao
{
    public class FrmCadastroManutencao : Form
    {
        private TextBox txtId, txtManutencao, txtObservacoes, txtDiaSemana;
        private ComboBox cboPlanta, cboMaquina, cboPeca, cboSituacao, cboFuncionario;
        private NumericUpDown numQuantidade, numPeriodicidade;
        private DateTimePicker dtpDataPlanejada, dtpDataRealizada;
        private CheckBox chkRealizada;
        private DataGridView dgvManutencoes;
        private Button btnNovo, btnSalvar, btnEditar, btnExcluir, btnCancelar, btnMarcarRealizado;
        private Panel pnlFormulario;
        private Label lblTituloPrincipal;

        public FrmCadastroManutencao()
        {
            CriarInterface();
            CarregarCombos();
            CarregarManutencoes();
            
            // Atualizar combos quando o formulário recebe foco
            this.Activated += FrmCadastroManutencao_Activated;
        }

        // Evento disparado quando o formulário é ativado (recebe foco)
        private void FrmCadastroManutencao_Activated(object sender, EventArgs e)
        {
            // Recarregar apenas se estiver no modo de visualização (botão Novo habilitado)
            if (btnNovo.Enabled)
            {
                CarregarCombos();
                CarregarManutencoes();
            }
        }

        // Construtor que recebe ID para edição automática
        public FrmCadastroManutencao(int idManutencao) : this()
        {
            // Aguardar o form carregar completamente
            this.Load += (s, e) => CarregarParaEdicao(idManutencao);
        }

        private void CarregarParaEdicao(int idManutencao)
        {
            try
            {
                // Procurar a linha no grid
                foreach (DataGridViewRow row in dgvManutencoes.Rows)
                {
                    if (row.Cells["ID"].Value != null && 
                        Convert.ToInt32(row.Cells["ID"].Value) == idManutencao)
                    {
                        // Selecionar a linha
                        dgvManutencoes.ClearSelection();
                        row.Selected = true;
                        dgvManutencoes.CurrentCell = row.Cells[0];
                        dgvManutencoes.FirstDisplayedScrollingRowIndex = row.Index;
                        
                        // Chamar o método de edição
                        BtnEditar_Click(null, null);
                        break;
                    }
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar manutenção para edição: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CriarInterface()
        {
            this.Text = "Cadastro de Manutenções";
            this.Size = new Size(1400, 850);
            this.StartPosition = FormStartPosition.CenterScreen;
            this.BackColor = Color.White;

            // Título Principal
            lblTituloPrincipal = new Label
            {
                Text = "🔧 Cadastro de Manutenções",
                Location = new Point(30, 20),
                Font = new Font("Segoe UI", 18, FontStyle.Bold),
                ForeColor = Color.FromArgb(41, 128, 185),
                AutoSize = true
            };

            // Painel do Formulário
            pnlFormulario = new Panel
            {
                Location = new Point(30, 70),
                Size = new Size(1320, 420),
                BackColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle
            };

            // Linha 1 - ID
            Label lblId = CriarLabel("ID:", 20, 25);
            txtId = new TextBox
            {
                Location = new Point(200, 22),
                Width = 100,
                Font = new Font("Segoe UI", 10),
                Enabled = false,
                BackColor = Color.FromArgb(236, 240, 241)
            };

            // Linha 2 - Planta e Máquina
            Label lblPlanta = CriarLabel("Planta: *", 20, 65, true);
            cboPlanta = CriarComboBox(200, 62, 180);
            // Plantas serão carregadas dinamicamente do banco de dados

            Label lblMaquina = CriarLabel("Máquina: *", 420, 65, true);
            cboMaquina = CriarComboBox(560, 62, 550);

            // Linha 3 - Peça e Quantidade
            Label lblPeca = CriarLabel("Peça: *", 20, 105, true);
            cboPeca = CriarComboBox(200, 102, 650);

            Label lblQtd = CriarLabel("Qtd: *", 890, 105, true);
            numQuantidade = new NumericUpDown
            {
                Location = new Point(950, 102),
                Width = 100,
                Font = new Font("Segoe UI", 10),
                Minimum = 1,
                Value = 1
            };

            // Linha 4 - Datas
            Label lblDataPlanejada = CriarLabel("Data Planejada: *", 20, 145, true);
            dtpDataPlanejada = new DateTimePicker
            {
                Location = new Point(200, 142),
                Width = 180,
                Format = DateTimePickerFormat.Short,
                Font = new Font("Segoe UI", 10)
            };
            dtpDataPlanejada.ValueChanged += (s, e) => AtualizarDiaSemana();

            Label lblDataRealizada = CriarLabel("Data Realizada:", 420, 145);
            dtpDataRealizada = new DateTimePicker
            {
                Location = new Point(560, 142),
                Width = 180,
                Format = DateTimePickerFormat.Short,
                Font = new Font("Segoe UI", 10),
                Enabled = false
            };

            chkRealizada = new CheckBox
            {
                Text = "✓ Realizado",
                Location = new Point(760, 145),
                Width = 150,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                ForeColor = Color.FromArgb(39, 174, 96)
            };
            chkRealizada.CheckedChanged += (s, e) => { dtpDataRealizada.Enabled = chkRealizada.Checked; };

            Label lblDiaSemana = CriarLabel("Dia Semana:", 950, 145);
            txtDiaSemana = new TextBox
            {
                Location = new Point(1070, 142),
                Width = 150,
                Font = new Font("Segoe UI", 10),
                ReadOnly = true,
                BackColor = Color.FromArgb(236, 240, 241)
            };

            // Linha 5 - Situação e Periodicidade
            Label lblSituacao = CriarLabel("Situação: *", 20, 185, true);
            cboSituacao = CriarComboBox(200, 182, 180);
            cboSituacao.Items.AddRange(new object[] { "PLANEJADO", "REALIZADO", "ATRASADO", "CANCELADO" });
            cboSituacao.SelectedIndex = 0;

            Label lblPeriodicidade = CriarLabel("Periodicidade (meses):", 420, 185);
            numPeriodicidade = new NumericUpDown
            {
                Location = new Point(600, 182),
                Width = 100,
                Font = new Font("Segoe UI", 10),
                Minimum = 1,
                Maximum = 60,
                Value = 6
            };

            Label lblInfoPeriod = new Label
            {
                Text = "ℹ️ Ex: 6 meses = semestral",
                Location = new Point(720, 185),
                Font = new Font("Segoe UI", 9, FontStyle.Italic),
                ForeColor = Color.FromArgb(127, 140, 141),
                AutoSize = true
            };

            // Linha 6 - Funcionário
            Label lblFunc = CriarLabel("Funcionário Responsável:", 20, 225);
            cboFuncionario = CriarComboBox(200, 222, 400);

            // Linha 7 - Descrição da Manutenção
            Label lblManutencao = CriarLabel("Descrição da Manutenção:", 20, 265);
            txtManutencao = new TextBox
            {
                Location = new Point(200, 262),
                Width = 1000,
                Height = 60,
                Font = new Font("Segoe UI", 10),
                Multiline = true,
                ScrollBars = ScrollBars.Vertical
            };

            // Linha 8 - Observações
            Label lblObs = CriarLabel("Observações:", 20, 335);
            txtObservacoes = new TextBox
            {
                Location = new Point(200, 332),
                Width = 1000,
                Height = 60,
                Font = new Font("Segoe UI", 10),
                Multiline = true,
                ScrollBars = ScrollBars.Vertical
            };

            // Informações
            Label lblObrigatorios = new Label
            {
                Text = "* Campos obrigatórios",
                Location = new Point(20, 400),
                Font = new Font("Segoe UI", 9, FontStyle.Bold),
                ForeColor = Color.FromArgb(192, 57, 43),
                AutoSize = true
            };

            pnlFormulario.Controls.AddRange(new Control[] {
                lblId, txtId, lblPlanta, cboPlanta, lblMaquina, cboMaquina,
                lblPeca, cboPeca, lblQtd, numQuantidade,
                lblDataPlanejada, dtpDataPlanejada, lblDataRealizada, dtpDataRealizada, chkRealizada,
                lblDiaSemana, txtDiaSemana, lblSituacao, cboSituacao,
                lblPeriodicidade, numPeriodicidade, lblInfoPeriod,
                lblFunc, cboFuncionario, lblManutencao, txtManutencao,
                lblObs, txtObservacoes, lblObrigatorios
            });

            // Botões
            int btnY = 510;
            btnNovo = CriarBotao("➕ Novo", Color.FromArgb(41, 128, 185), 30, btnY, 150);
            btnNovo.Click += BtnNovo_Click;

            btnSalvar = CriarBotao("💾 Salvar", Color.FromArgb(39, 174, 96), 190, btnY, 130);
            btnSalvar.Click += BtnSalvar_Click;

            btnEditar = CriarBotao("✏️ Editar", Color.FromArgb(243, 156, 18), 330, btnY, 130);
            btnEditar.Click += BtnEditar_Click;

            btnMarcarRealizado = CriarBotao("✅ Marcar Realizado", Color.FromArgb(39, 174, 96), 470, btnY, 180);
            btnMarcarRealizado.Click += BtnMarcarRealizado_Click;

            btnExcluir = CriarBotao("🗑️ Excluir", Color.FromArgb(192, 57, 43), 660, btnY, 130);
            btnExcluir.Click += BtnExcluir_Click;

            btnCancelar = CriarBotao("❌ Cancelar", Color.FromArgb(127, 140, 141), 800, btnY, 130);
            btnCancelar.Click += BtnCancelar_Click;

            // DataGridView
            dgvManutencoes = CriarDataGrid(30, 570, 1320, 230);
            dgvManutencoes.CellClick += DgvManutencoes_CellClick;
            dgvManutencoes.CellFormatting += DgvManutencoes_CellFormatting;

            this.Controls.AddRange(new Control[] {
                lblTituloPrincipal, pnlFormulario, btnNovo, btnSalvar, btnEditar,
                btnMarcarRealizado, btnExcluir, btnCancelar, dgvManutencoes
            });

            ConfigurarBotoes(true);
        }

        private void DgvManutencoes_CellFormatting(object sender, DataGridViewCellFormattingEventArgs e)
        {
            if (dgvManutencoes.Columns[e.ColumnIndex].Name == "Status")
            {
                string status = e.Value?.ToString();

                if (status == "ATRASADO")
                {
                    e.CellStyle.ForeColor = Color.Red;
                    e.CellStyle.Font = new Font(dgvManutencoes.Font, FontStyle.Bold);
                }
                else if (status == "REALIZADO")
                {
                    e.CellStyle.ForeColor = Color.Green;
                    e.CellStyle.Font = new Font(dgvManutencoes.Font, FontStyle.Bold);
                }
                else if (status == "HOJE")
                {
                    e.CellStyle.ForeColor = Color.DarkGoldenrod;
                    e.CellStyle.Font = new Font(dgvManutencoes.Font, FontStyle.Bold);
                }
                else if (status == "PLANEJADO")
                {
                    e.CellStyle.ForeColor = Color.Blue;
                }
            }
        }

        private void DgvManutencoes_CellClick(object sender, DataGridViewCellEventArgs e)
        {
            if (e.RowIndex < 0) return;

            DataGridViewRow row = dgvManutencoes.Rows[e.RowIndex];
            txtId.Text = row.Cells["ID"].Value.ToString();
            ConfigurarBotoes(true);
        }


        private Label CriarLabel(string texto, int x, int y, bool negrito = false)
        {
            return new Label
            {
                Text = texto,
                Location = new Point(x, y),
                Font = new Font("Segoe UI", 10, negrito ? FontStyle.Bold : FontStyle.Regular),
                AutoSize = true
            };
        }

        private void InitializeComponent()
        {
            this.SuspendLayout();
            // 
            // FrmCadastroManutencao
            // 
            this.ClientSize = new System.Drawing.Size(284, 261);
            this.Name = "FrmCadastroManutencao";
            this.Load += new System.EventHandler(this.FrmCadastroManutencao_Load);
            this.ResumeLayout(false);

        }

        private void FrmCadastroManutencao_Load(object sender, EventArgs e)
        {

        }

        private ComboBox CriarComboBox(int x, int y, int largura)
        {
            return new ComboBox
            {
                Location = new Point(x, y),
                Width = largura,
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };
        }

        private Button CriarBotao(string texto, Color cor, int x, int y, int largura)
        {
            Button btn = new Button
            {
                Text = texto,
                Location = new Point(x, y),
                Width = largura,
                Height = 45,
                FlatStyle = FlatStyle.Flat,
                BackColor = cor,
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                Cursor = Cursors.Hand
            };
            btn.FlatAppearance.BorderSize = 0;
            return btn;
        }

        private DataGridView CriarDataGrid(int x, int y, int largura, int altura)
        {
            DataGridView dgv = new DataGridView
            {
                Location = new Point(x, y),
                Size = new Size(largura, altura),
                AllowUserToAddRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                ColumnHeadersDefaultCellStyle = new DataGridViewCellStyle
                {
                    BackColor = Color.FromArgb(52, 73, 94),
                    ForeColor = Color.White,
                    Font = new Font("Segoe UI", 9, FontStyle.Bold)
                },
                EnableHeadersVisualStyles = false,
                RowHeadersVisible = false
            };
            return dgv;
        }

        private void AtualizarDiaSemana()
        {
            string[] dias = { "domingo", "segunda-feira", "terça-feira", "quarta-feira", "quinta-feira", "sexta-feira", "sábado" };
            txtDiaSemana.Text = dias[(int)dtpDataPlanejada.Value.DayOfWeek];
        }

        private void ConfigurarBotoes(bool novo)
        {
            btnNovo.Enabled = novo;
            btnSalvar.Enabled = !novo;
            btnCancelar.Enabled = !novo;
            btnEditar.Enabled = novo && dgvManutencoes.SelectedRows.Count > 0;
            btnExcluir.Enabled = novo && dgvManutencoes.SelectedRows.Count > 0;
            btnMarcarRealizado.Enabled = novo && dgvManutencoes.SelectedRows.Count > 0;
        }

        private void CarregarCombos()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // Carregar Plantas dinamicamente do banco de dados
                    MySqlCommand cmdPlanta = new MySqlCommand("SELECT DISTINCT planta FROM maquinas ORDER BY planta", conn);
                    MySqlDataReader readerPlanta = cmdPlanta.ExecuteReader();
                    cboPlanta.Items.Clear();
                    while (readerPlanta.Read())
                    {
                        cboPlanta.Items.Add(readerPlanta.GetString(0));
                    }
                    readerPlanta.Close();

                    // Carregar Máquinas
                    MySqlCommand cmdMaq = new MySqlCommand("SELECT id_maquina, nome_maquina FROM maquinas ORDER BY nome_maquina", conn);
                    MySqlDataReader reader = cmdMaq.ExecuteReader();
                    cboMaquina.Items.Clear();
                    while (reader.Read())
                    {
                        cboMaquina.Items.Add(new { Id = reader.GetInt32(0), Nome = reader.GetString(1) });
                    }
                    cboMaquina.DisplayMember = "Nome";
                    cboMaquina.ValueMember = "Id";
                    reader.Close();

                    // Carregar Peças
                    MySqlCommand cmdPeca = new MySqlCommand("SELECT id_peca, nome_peca FROM pecas ORDER BY nome_peca", conn);
                    reader = cmdPeca.ExecuteReader();
                    cboPeca.Items.Clear();
                    while (reader.Read())
                    {
                        cboPeca.Items.Add(new { Id = reader.GetInt32(0), Nome = reader.GetString(1) });
                    }
                    cboPeca.DisplayMember = "Nome";
                    cboPeca.ValueMember = "Id";
                    reader.Close();

                    // Carregar Funcionários
                    MySqlCommand cmdFunc = new MySqlCommand("SELECT id_funcionario, nome FROM funcionarios WHERE status='ativo' ORDER BY nome", conn);
                    reader = cmdFunc.ExecuteReader();
                    cboFuncionario.Items.Clear();
                    while (reader.Read())
                    {
                        cboFuncionario.Items.Add(new { Id = reader.GetInt32(0), Nome = reader.GetString(1) });
                    }
                    cboFuncionario.DisplayMember = "Nome";
                    cboFuncionario.ValueMember = "Id";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro ao carregar dados: " + ex.Message, "Erro", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnNovo_Click(object sender, EventArgs e)
        {
            // Recarregar combos para pegar novos registros
            CarregarCombos();
            
            LimparCampos();
            ConfigurarBotoes(false);
            cboPlanta.Focus();
            AtualizarDiaSemana();
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            // Validação
            if (cboPlanta.SelectedIndex == -1)
            {
                MessageBox.Show("❌ Selecione a Planta!", "Campo Obrigatório", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                cboPlanta.Focus();
                return;
            }

            if (cboMaquina.SelectedIndex == -1)
            {
                MessageBox.Show("❌ Selecione a Máquina!", "Campo Obrigatório", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                cboMaquina.Focus();
                return;
            }

            if (cboPeca.SelectedIndex == -1)
            {
                MessageBox.Show("❌ Selecione a Peça!", "Campo Obrigatório", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                cboPeca.Focus();
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    dynamic maquina = cboMaquina.SelectedItem;
                    dynamic peca = cboPeca.SelectedItem;
                    dynamic funcionario = cboFuncionario.SelectedIndex >= 0 ? cboFuncionario.SelectedItem : null;

                    int periodicidadeMeses = (int)numPeriodicidade.Value;

                    // Calcular próxima manutenção (se foi realizada)
                    DateTime? proximaManutencao = null;
                    if (chkRealizada.Checked && dtpDataRealizada.Enabled)
                    {
                        proximaManutencao = dtpDataRealizada.Value.AddMonths(periodicidadeMeses);
                    }

                    string query = string.IsNullOrEmpty(txtId.Text)
                        ? @"INSERT INTO manutencoes (planta, id_maquina, id_peca, quantidade, data_planejada, 
                            data_realizada, situacao, id_funcionario, descricao_manutencao, dia_semana, 
                            observacoes, periodicidade_meses, proxima_manutencao) 
                            VALUES (@planta, @maq, @peca, @qtd, @dtplan, @dtreal, @sit, @func, @manut, @dia, @obs, @period, @proxima)"
                        : @"UPDATE manutencoes SET planta=@planta, id_maquina=@maq, id_peca=@peca, quantidade=@qtd,
                            data_planejada=@dtplan, data_realizada=@dtreal, situacao=@sit,
                            id_funcionario=@func, descricao_manutencao=@manut, dia_semana=@dia, observacoes=@obs, 
                            periodicidade_meses=@period, proxima_manutencao=@proxima WHERE id_manutencao=@id";

                    using (MySqlCommand cmd = new MySqlCommand(query, conn))
                    {
                        if (!string.IsNullOrEmpty(txtId.Text))
                            cmd.Parameters.AddWithValue("@id", txtId.Text);

                        cmd.Parameters.AddWithValue("@planta", cboPlanta.SelectedItem.ToString());
                        cmd.Parameters.AddWithValue("@maq", maquina.Id);
                        cmd.Parameters.AddWithValue("@peca", peca.Id);
                        cmd.Parameters.AddWithValue("@qtd", numQuantidade.Value);
                        cmd.Parameters.AddWithValue("@dtplan", dtpDataPlanejada.Value.Date);
                        cmd.Parameters.AddWithValue("@dtreal", dtpDataRealizada.Enabled ? (object)dtpDataRealizada.Value.Date : DBNull.Value);
                        cmd.Parameters.AddWithValue("@sit", cboSituacao.SelectedItem.ToString());

                        // Funcionário
                        if (funcionario != null)
                            cmd.Parameters.AddWithValue("@func", funcionario.Id);
                        else
                            cmd.Parameters.AddWithValue("@func", DBNull.Value);

                        // Descrição
                        if (string.IsNullOrWhiteSpace(txtManutencao.Text))
                            cmd.Parameters.AddWithValue("@manut", DBNull.Value);
                        else
                            cmd.Parameters.AddWithValue("@manut", txtManutencao.Text);

                        cmd.Parameters.AddWithValue("@dia", txtDiaSemana.Text);

                        // Observações
                        if (string.IsNullOrWhiteSpace(txtObservacoes.Text))
                            cmd.Parameters.AddWithValue("@obs", DBNull.Value);
                        else
                            cmd.Parameters.AddWithValue("@obs", txtObservacoes.Text);

                        // Periodicidade
                        cmd.Parameters.AddWithValue("@period", periodicidadeMeses);

                        // Próxima manutenção
                        if (proximaManutencao.HasValue)
                            cmd.Parameters.AddWithValue("@proxima", proximaManutencao.Value);
                        else
                            cmd.Parameters.AddWithValue("@proxima", DBNull.Value);

                        cmd.ExecuteNonQuery();
                    }
                }
                
                string mensagemSucesso = "✅ Manutenção salva com sucesso!";

                // Calcular próxima manutenção para mostrar
                if (chkRealizada.Checked && dtpDataRealizada.Enabled)
                {
                    DateTime proxima = dtpDataRealizada.Value.AddMonths((int)numPeriodicidade.Value);
                    mensagemSucesso += $"\n\n📅 Próxima manutenção agendada para: {proxima:dd/MM/yyyy}";
                }

                MessageBox.Show(mensagemSucesso, "Sucesso", MessageBoxButtons.OK, MessageBoxIcon.Information);
                CarregarManutencoes();
                LimparCampos();
                ConfigurarBotoes(true);
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro: " + ex.Message, "Erro", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnEditar_Click(object sender, EventArgs e)
        {
            if (dgvManutencoes.SelectedRows.Count == 0) return;

            try
            {
                DataGridViewRow row = dgvManutencoes.SelectedRows[0];
                txtId.Text = row.Cells["ID"].Value.ToString();

                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    MySqlCommand cmd = new MySqlCommand(@"SELECT m.*, maq.nome_maquina, p.nome_peca, f.nome as nome_func
                        FROM manutencoes m
                        INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
                        INNER JOIN pecas p ON m.id_peca = p.id_peca
                        LEFT JOIN funcionarios f ON m.id_funcionario = f.id_funcionario
                        WHERE id_manutencao = @id", conn);
                    cmd.Parameters.AddWithValue("@id", txtId.Text);

                    MySqlDataReader reader = cmd.ExecuteReader();
                    if (reader.Read())
                    {
                        cboPlanta.SelectedItem = reader["planta"].ToString();

                        // Selecionar máquina
                        for (int i = 0; i < cboMaquina.Items.Count; i++)
                        {
                            dynamic item = cboMaquina.Items[i];
                            if (item.Id == Convert.ToInt32(reader["id_maquina"]))
                            {
                                cboMaquina.SelectedIndex = i;
                                break;
                            }
                        }

                        // Selecionar peça
                        for (int i = 0; i < cboPeca.Items.Count; i++)
                        {
                            dynamic item = cboPeca.Items[i];
                            if (item.Id == Convert.ToInt32(reader["id_peca"]))
                            {
                                cboPeca.SelectedIndex = i;
                                break;
                            }
                        }

                        numQuantidade.Value = Convert.ToInt32(reader["quantidade"]);
                        dtpDataPlanejada.Value = reader.GetDateTime(reader.GetOrdinal("data_planejada"));

                        if (reader["data_realizada"] != DBNull.Value)
                        {
                            dtpDataRealizada.Value = reader.GetDateTime(reader.GetOrdinal("data_realizada"));
                            chkRealizada.Checked = true;
                        }

                        cboSituacao.SelectedItem = reader["situacao"].ToString();

                        // Periodicidade
                        if (reader["periodicidade_meses"] != DBNull.Value)
                            numPeriodicidade.Value = Convert.ToInt32(reader["periodicidade_meses"]);

                        // Selecionar funcionário
                        if (reader["id_funcionario"] != DBNull.Value)
                        {
                            for (int i = 0; i < cboFuncionario.Items.Count; i++)
                            {
                                dynamic item = cboFuncionario.Items[i];
                                if (item.Id == Convert.ToInt32(reader["id_funcionario"]))
                                {
                                    cboFuncionario.SelectedIndex = i;
                                    break;
                                }
                            }
                        }

                        txtManutencao.Text = reader["descricao_manutencao"]?.ToString();
                        txtDiaSemana.Text = reader["dia_semana"]?.ToString();
                        txtObservacoes.Text = reader["observacoes"]?.ToString();
                    }
                }

                ConfigurarBotoes(false);
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro: " + ex.Message, "Erro", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnMarcarRealizado_Click(object sender, EventArgs e)
        {
            if (dgvManutencoes.SelectedRows.Count == 0) return;

            int id = Convert.ToInt32(dgvManutencoes.SelectedRows[0].Cells["ID"].Value);

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string sql = @"UPDATE manutencoes 
                           SET situacao='REALIZADO', 
                               data_realizada=@hoje 
                           WHERE id_manutencao=@id";

                    MySqlCommand cmd = new MySqlCommand(sql, conn);
                    cmd.Parameters.AddWithValue("@hoje", DateTime.Now);
                    cmd.Parameters.AddWithValue("@id", id);
                    cmd.ExecuteNonQuery();
                }

                MessageBox.Show("Manutenção marcada como realizada!");
                CarregarManutencoes();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro: " + ex.Message);
            }
        }

        private void BtnExcluir_Click(object sender, EventArgs e)
        {
            if (dgvManutencoes.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma manutenção para excluir.");
                return;
            }

            int id = Convert.ToInt32(dgvManutencoes.SelectedRows[0].Cells["ID"].Value);

            if (MessageBox.Show("Deseja realmente excluir esta manutenção?",
                "Confirmar Exclusão", MessageBoxButtons.YesNo) == DialogResult.No) return;

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    MySqlCommand cmd = new MySqlCommand("DELETE FROM manutencoes WHERE id_manutencao=@id", conn);
                    cmd.Parameters.AddWithValue("@id", id);
                    cmd.ExecuteNonQuery();
                }

                MessageBox.Show("Manutenção excluída com sucesso!");
                CarregarManutencoes();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro ao excluir: " + ex.Message);
            }
        }


        private void BtnCancelar_Click(object sender, EventArgs e)
        {
            LimparCampos();
            ConfigurarBotoes(true);
        }


        private void LimparCampos()
        {
            txtId.Clear();
            cboPlanta.SelectedIndex = -1;
            cboMaquina.SelectedIndex = -1;
            cboPeca.SelectedIndex = -1;
            cboFuncionario.SelectedIndex = -1;
            numQuantidade.Value = 1;
            dtpDataPlanejada.Value = DateTime.Now;
            dtpDataRealizada.Value = DateTime.Now;
            chkRealizada.Checked = false;
            cboSituacao.SelectedIndex = 0;
            numPeriodicidade.Value = 6;
            txtManutencao.Clear();
            txtObservacoes.Clear();
            txtDiaSemana.Clear();
        }


        private void CarregarManutencoes()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string sql = @"SELECT 
                m.id_manutencao AS ID,
                m.planta AS Planta,
                maq.nome_maquina AS Maquina,
                p.nome_peca AS Peca,
                m.quantidade AS Quantidade,
                DATE_FORMAT(m.data_planejada, '%d/%m/%Y') AS Planejada,
                DATE_FORMAT(m.data_realizada, '%d/%m/%Y') AS Realizada,
                m.situacao AS StatusBanco,
                m.dia_semana AS Dia,
                m.periodicidade_meses AS Periodicidade,
                DATE_FORMAT(m.proxima_manutencao, '%d/%m/%Y') AS Próxima,
                f.nome AS Funcionário,
                m.descricao_manutencao AS Descrição,
                m.observacoes AS Observações
            FROM manutencoes m
            INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
            INNER JOIN pecas p ON m.id_peca = p.id_peca
            LEFT JOIN funcionarios f ON m.id_funcionario = f.id_funcionario
            ORDER BY m.data_planejada DESC";

                    MySqlDataAdapter da = new MySqlDataAdapter(sql, conn);
                    DataTable dt = new DataTable();
                    da.Fill(dt);

                    // ============================================
                    // Criar coluna calculada "Status"
                    // ============================================
                    dt.Columns.Add("Status", typeof(string));
                    DateTime hoje = DateTime.Today;

                    foreach (DataRow row in dt.Rows)
                    {
                        string statusBanco = row["StatusBanco"]?.ToString() ?? "";
                        DateTime? dataPlanejada = null;
                        if (row["Planejada"] != DBNull.Value && !string.IsNullOrEmpty(row["Planejada"].ToString()))
                        {
                            try
                            {
                                dataPlanejada = DateTime.ParseExact(row["Planejada"].ToString(), "dd/MM/yyyy", null);
                            }
                            catch { }
                        }

                        string statusFinal = statusBanco;

                        // Lógica de status dinâmico
                        if (statusBanco == "REALIZADO")
                        {
                            statusFinal = "REALIZADO";
                        }
                        else if (dataPlanejada.HasValue)
                        {
                            if (dataPlanejada.Value.Date < hoje)
                                statusFinal = "ATRASADO";
                            else if (dataPlanejada.Value.Date == hoje)
                                statusFinal = "HOJE";
                            else
                                statusFinal = "PLANEJADO";
                        }

                        row["Status"] = statusFinal;
                    }

                    dgvManutencoes.DataSource = dt;

                    // Ocultar colunas auxiliares
                    if (dgvManutencoes.Columns["Planejada"] != null)
                        dgvManutencoes.Columns["Planejada"].Visible = false;

                    if (dgvManutencoes.Columns["StatusBanco"] != null)
                        dgvManutencoes.Columns["StatusBanco"].Visible = false;

                    // Definir ordem das colunas
                    int idx = 0;
                    if (dgvManutencoes.Columns["ID"] != null)
                        dgvManutencoes.Columns["ID"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Planta"] != null)
                        dgvManutencoes.Columns["Planta"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Maquina"] != null)
                        dgvManutencoes.Columns["Maquina"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Peca"] != null)
                        dgvManutencoes.Columns["Peca"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Quantidade"] != null)
                        dgvManutencoes.Columns["Quantidade"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Planejada"] != null)
                        dgvManutencoes.Columns["Planejada"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Realizada"] != null)
                        dgvManutencoes.Columns["Realizada"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Status"] != null)
                        dgvManutencoes.Columns["Status"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Dia"] != null)
                        dgvManutencoes.Columns["Dia"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Periodicidade"] != null)
                        dgvManutencoes.Columns["Periodicidade"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Próxima"] != null)
                        dgvManutencoes.Columns["Próxima"].DisplayIndex = idx++;
                    if (dgvManutencoes.Columns["Funcionário"] != null)
                        dgvManutencoes.Columns["Funcionário"].DisplayIndex = idx++;

                    // Aplicar cores às linhas com base no Status
                    foreach (DataGridViewRow row in dgvManutencoes.Rows)
                    {
                        if (row.Cells["Status"]?.Value == null) continue;

                        string status = row.Cells["Status"].Value.ToString();

                        if (status == "REALIZADO")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(230, 255, 230); // Verde claro
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(0, 100, 0);     // Verde escuro
                        }
                        else if (status == "ATRASADO")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(255, 230, 230); // Vermelho claro
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(139, 0, 0);     // Vermelho escuro
                        }
                        else if (status == "HOJE")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(255, 255, 200); // Amarelo claro
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(139, 90, 0);    // Marrom
                            row.DefaultCellStyle.Font = new Font(dgvManutencoes.Font, FontStyle.Bold);
                        }
                        else if (status == "PLANEJADO")
                        {
                            row.DefaultCellStyle.BackColor = Color.FromArgb(220, 235, 255); // Azul claro
                            row.DefaultCellStyle.ForeColor = Color.FromArgb(0, 51, 102);    // Azul escuro
                        }
                    }

                    dgvManutencoes.Refresh();
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("Erro ao carregar manutenções: " + ex.Message, "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

    }
}
using System;
using System.Drawing;
using System.Windows.Forms;

namespace SistemaManutencao
{
    public class FrmConfiguracoes : Form
    {
        private Panel pnlConfiguracoes;
        private GroupBox grpWhatsApp;
        private Label lblGrupoId, lblDiasAntecedencia;
        private TextBox txtGrupoId, txtDiasAntecedencia;
        private Button btnSalvar, btnTestar, btnAjuda;
        private Label lblStatus;
        private CheckBox chkSistemaAtivo;

        public FrmConfiguracoes()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarConfiguracoes();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Configurações do Sistema";
            this.Size = new Size(900, 700);
            this.StartPosition = FormStartPosition.CenterScreen;
            this.BackColor = Color.White;
            this.FormBorderStyle = FormBorderStyle.FixedDialog;
            this.MaximizeBox = false;
        }

        private void CriarInterface()
        {
            // Título
            Label lblTitulo = new Label
            {
                Text = "⚙️ Configurações do Sistema",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // Painel de Configurações
            pnlConfiguracoes = new Panel
            {
                Location = new Point(30, 80),
                Size = new Size(820, 550),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle,
                AutoScroll = true
            };

            // Grupo WhatsApp
            grpWhatsApp = new GroupBox
            {
                Text = "📱 WhatsApp Web Automático",
                Location = new Point(20, 20),
                Size = new Size(760, 500),
                Font = new Font("Segoe UI", 12, FontStyle.Bold),
                ForeColor = Color.FromArgb(0, 123, 255)
            };

            int y = 40;

            // Sistema Ativo
            chkSistemaAtivo = new CheckBox
            {
                Text = "✅ Sistema de Alertas ATIVO",
                Location = new Point(20, y),
                Size = new Size(300, 25),
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                ForeColor = Color.FromArgb(40, 167, 69),
                Checked = true
            };
            y += 50;

            // Número WhatsApp
            lblGrupoId = new Label
            {
                Text = "📱 Número do WhatsApp (contato ou grupo):",
                Location = new Point(20, y),
                Size = new Size(700, 25),
                Font = new Font("Segoe UI", 10)
            };
            y += 30;

            txtGrupoId = new TextBox
            {
                Location = new Point(20, y),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 12),
                Text = "5519989102583"
            };
            y += 40;

            Label lblFormato = new Label
            {
                Text = "Formato: 5519989102583 (país + DDD + número, sem espaços)",
                Location = new Point(20, y),
                Size = new Size(700, 20),
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.Gray
            };
            y += 40;

            // Dias antecedência
            lblDiasAntecedencia = new Label
            {
                Text = "⏰ Dias de antecedência para alertas:",
                Location = new Point(20, y),
                Size = new Size(300, 25),
                Font = new Font("Segoe UI", 10)
            };
            y += 30;

            txtDiasAntecedencia = new TextBox
            {
                Location = new Point(20, y),
                Size = new Size(80, 30),
                Font = new Font("Segoe UI", 12),
                Text = "3",
                MaxLength = 2
            };

            Label lblDias = new Label
            {
                Text = "dias",
                Location = new Point(110, y + 5),
                Size = new Size(50, 20),
                Font = new Font("Segoe UI", 10)
            };
            y += 60;

            // Informações
            Panel pnlInfo = new Panel
            {
                Location = new Point(20, y),
                Size = new Size(700, 100),
                BackColor = Color.FromArgb(255, 243, 205),
                BorderStyle = BorderStyle.FixedSingle
            };

            Label lblInfo = new Label
            {
                Text = "ℹ️ COMO FUNCIONA:\n" +
                       "• Sistema abre WhatsApp Web automaticamente\n" +
                       "• Aguarda 25 segundos para carregar\n" +
                       "• Pressiona Enter automaticamente\n" +
                       "• 100% automático - sem necessidade de clicar!",
                Location = new Point(10, 10),
                Size = new Size(680, 80),
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.FromArgb(133, 100, 4)
            };
            pnlInfo.Controls.Add(lblInfo);
            y += 120;

            // Botões
            btnSalvar = new Button
            {
                Text = "💾 Salvar Configurações",
                Location = new Point(20, y),
                Size = new Size(200, 45),
                BackColor = Color.FromArgb(40, 167, 69),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnSalvar.FlatAppearance.BorderSize = 0;
            btnSalvar.Click += BtnSalvar_Click;

            btnTestar = new Button
            {
                Text = "🧪 Testar Envio",
                Location = new Point(240, y),
                Size = new Size(200, 45),
                BackColor = Color.FromArgb(0, 123, 255),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnTestar.FlatAppearance.BorderSize = 0;
            btnTestar.Click += BtnTestar_Click;

            btnAjuda = new Button
            {
                Text = "❓ Ajuda",
                Location = new Point(460, y),
                Size = new Size(150, 45),
                BackColor = Color.FromArgb(255, 193, 7),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAjuda.FlatAppearance.BorderSize = 0;
            btnAjuda.Click += BtnAjuda_Click;
            y += 60;

            // Status
            lblStatus = new Label
            {
                Location = new Point(20, y),
                Size = new Size(700, 30),
                Font = new Font("Segoe UI", 10),
                ForeColor = Color.Gray,
                Text = "Configure o número e teste o envio automático"
            };

            grpWhatsApp.Controls.AddRange(new Control[] {
                chkSistemaAtivo,
                lblGrupoId, txtGrupoId, lblFormato,
                lblDiasAntecedencia, txtDiasAntecedencia, lblDias,
                pnlInfo,
                btnSalvar, btnTestar, btnAjuda,
                lblStatus
            });

            pnlConfiguracoes.Controls.Add(grpWhatsApp);
            this.Controls.AddRange(new Control[] { lblTitulo, pnlConfiguracoes });
        }

        private void InitializeComponent()
        {
            this.SuspendLayout();
            // 
            // FrmConfiguracoes
            // 
            this.ClientSize = new System.Drawing.Size(284, 261);
            this.Name = "FrmConfiguracoes";
            this.Load += new System.EventHandler(this.FrmConfiguracoes_Load);
            this.ResumeLayout(false);

        }

        private void FrmConfiguracoes_Load(object sender, EventArgs e)
        {

        }

        private void CarregarConfiguracoes()
        {
            try
            {
                string grupoId = ConexaoDB.ObterConfiguracao("whatsapp_grupo_id", "5519989102583");
                if (string.IsNullOrWhiteSpace(grupoId))
                    grupoId = "5519989102583";
                txtGrupoId.Text = grupoId;

                string dias = ConexaoDB.ObterConfiguracao("dias_antecedencia_alerta", "3");
                txtDiasAntecedencia.Text = dias;

                string sistemaAtivo = ConexaoDB.ObterConfiguracao("sistema_ativo", "true");
                chkSistemaAtivo.Checked = sistemaAtivo.ToLower() == "true";
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar: {ex.Message}", "Erro", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            try
            {
                if (string.IsNullOrWhiteSpace(txtGrupoId.Text))
                {
                    MessageBox.Show("⚠️ Informe o número do WhatsApp!", 
                        "Validação", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                    txtGrupoId.Focus();
                    return;
                }

                if (!int.TryParse(txtDiasAntecedencia.Text, out int dias) || dias < 1 || dias > 30)
                {
                    MessageBox.Show("⚠️ Dias deve ser entre 1 e 30!", 
                        "Validação", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                    txtDiasAntecedencia.Focus();
                    return;
                }

                string numeroLimpo = txtGrupoId.Text.Replace("-", "").Replace(" ", "")
                    .Replace("(", "").Replace(")", "").Replace("+", "");

                ConexaoDB.SalvarConfiguracao("whatsapp_grupo_id", numeroLimpo);
                ConexaoDB.SalvarConfiguracao("dias_antecedencia_alerta", dias.ToString());
                ConexaoDB.SalvarConfiguracao("sistema_ativo", chkSistemaAtivo.Checked.ToString().ToLower());

                lblStatus.Text = "✅ Configurações salvas com sucesso!";
                lblStatus.ForeColor = Color.FromArgb(40, 167, 69);

                MessageBox.Show("✅ Configurações salvas!", "Sucesso", 
                    MessageBoxButtons.OK, MessageBoxIcon.Information);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"❌ Erro: {ex.Message}", "Erro", 
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnTestar_Click(object sender, EventArgs e)
        {
            try
            {
                btnTestar.Enabled = false;
                btnTestar.Text = "⏳ Enviando...";
                lblStatus.Text = "📱 Abrindo WhatsApp Web...";
                lblStatus.ForeColor = Color.FromArgb(0, 123, 255);
                Application.DoEvents();

                string numeroLimpo = txtGrupoId.Text.Replace("-", "").Replace(" ", "")
                    .Replace("(", "").Replace(")", "").Replace("+", "");

                ConexaoDB.SalvarConfiguracao("whatsapp_grupo_id", numeroLimpo);

                string msg = $"🧪 TESTE DO SISTEMA\n\n" +
                            $"WhatsApp configurado!\n" +
                            $"{DateTime.Now:dd/MM/yyyy HH:mm:ss}";

                ServicoWhatsApp.EnviarMensagem(numeroLimpo, msg);

                lblStatus.Text = "✅ WhatsApp aberto! Aguarde 25s...";
                lblStatus.ForeColor = Color.FromArgb(40, 167, 69);
            }
            catch (Exception ex)
            {
                lblStatus.Text = $"❌ Erro: {ex.Message}";
                lblStatus.ForeColor = Color.Red;
            }
            finally
            {
                btnTestar.Enabled = true;
                btnTestar.Text = "🧪 Testar Envio";
            }
        }

        private void BtnAjuda_Click(object sender, EventArgs e)
        {
            MessageBox.Show(
                "📱 CONFIGURAÇÃO DO WHATSAPP\n\n" +
                "CONTATO INDIVIDUAL:\n" +
                "• Formato: 5519989102583\n" +
                "• País (55) + DDD (19) + Número\n" +
                "• Sem espaços ou caracteres especiais\n\n" +
                "GRUPO:\n" +
                "• Abra o grupo no WhatsApp Web\n" +
                "• Clique no nome do grupo\n" +
                "• Role até o final\n" +
                "• Copie o ID do grupo\n\n" +
                "FUNCIONAMENTO:\n" +
                "• Sistema abre WhatsApp Web\n" +
                "• Aguarda 25 segundos\n" +
                "• Envia automaticamente\n" +
                "• 100% automático!",
                "Ajuda - Configurações",
                MessageBoxButtons.OK,
                MessageBoxIcon.Information
            );
        }
    }
}

using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmConsultaGeral : Form
    {
        private TabControl tabControl;
        private TabPage tabMaquinas, tabPecas, tabFuncionarios;
        private DataGridView dgvMaquinas, dgvPecas, dgvFuncionarios;
        private TextBox txtBuscaMaquina, txtBuscaPeca, txtBuscaFuncionario;
        private Label lblTitulo;

        public FrmConsultaGeral()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarDados();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Consulta Geral - Sistema de Manutenção";
            this.Size = new Size(1250, 750);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // Título
            lblTitulo = new Label
            {
                Text = "🔍 Consulta Geral",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // TabControl
            tabControl = new TabControl
            {
                Location = new Point(30, 80),
                Size = new Size(1170, 620),
                Font = new Font("Segoe UI", 10)
            };

            // Criar abas
            CriarAbaMaquinas();
            CriarAbaPecas();
            CriarAbaFuncionarios();

            this.Controls.AddRange(new Control[] { lblTitulo, tabControl });
        }

        #region ABA MÁQUINAS

        private void CriarAbaMaquinas()
        {
            tabMaquinas = new TabPage("⚙️ Máquinas");

            // Campo de busca
            Label lblBuscaMaquina = new Label
            {
                Text = "🔍 Buscar:",
                Location = new Point(20, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtBuscaMaquina = new TextBox
            {
                Location = new Point(110, 20),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };
            txtBuscaMaquina.TextChanged += (s, e) => FiltrarMaquinas();

            Button btnAtualizarMaquinas = new Button
            {
                Text = "🔄 Atualizar",
                Location = new Point(530, 15),
                Size = new Size(120, 35),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAtualizarMaquinas.FlatAppearance.BorderSize = 0;
            btnAtualizarMaquinas.Click += (s, e) => CarregarMaquinas();

            // DataGridView
            dgvMaquinas = CriarDataGridView();
            dgvMaquinas.Location = new Point(20, 70);
            dgvMaquinas.Size = new Size(1120, 500);

            tabMaquinas.Controls.AddRange(new Control[] { 
                lblBuscaMaquina, txtBuscaMaquina, btnAtualizarMaquinas, dgvMaquinas 
            });
            tabControl.TabPages.Add(tabMaquinas);
        }

        #endregion

        #region ABA PEÇAS

        private void CriarAbaPecas()
        {
            tabPecas = new TabPage("🔧 Peças");

            // Campo de busca
            Label lblBuscaPeca = new Label
            {
                Text = "🔍 Buscar:",
                Location = new Point(20, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtBuscaPeca = new TextBox
            {
                Location = new Point(110, 20),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };
            txtBuscaPeca.TextChanged += (s, e) => FiltrarPecas();

            Button btnAtualizarPecas = new Button
            {
                Text = "🔄 Atualizar",
                Location = new Point(530, 15),
                Size = new Size(120, 35),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAtualizarPecas.FlatAppearance.BorderSize = 0;
            btnAtualizarPecas.Click += (s, e) => CarregarPecas();

            // DataGridView
            dgvPecas = CriarDataGridView();
            dgvPecas.Location = new Point(20, 70);
            dgvPecas.Size = new Size(1120, 500);

            tabPecas.Controls.AddRange(new Control[] { 
                lblBuscaPeca, txtBuscaPeca, btnAtualizarPecas, dgvPecas 
            });
            tabControl.TabPages.Add(tabPecas);
        }

        #endregion

        #region ABA FUNCIONÁRIOS

        private void CriarAbaFuncionarios()
        {
            tabFuncionarios = new TabPage("👤 Funcionários");

            // Campo de busca
            Label lblBuscaFuncionario = new Label
            {
                Text = "🔍 Buscar:",
                Location = new Point(20, 20),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtBuscaFuncionario = new TextBox
            {
                Location = new Point(110, 20),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };
            txtBuscaFuncionario.TextChanged += (s, e) => FiltrarFuncionarios();

            Button btnAtualizarFuncionarios = new Button
            {
                Text = "🔄 Atualizar",
                Location = new Point(530, 15),
                Size = new Size(120, 35),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAtualizarFuncionarios.FlatAppearance.BorderSize = 0;
            btnAtualizarFuncionarios.Click += (s, e) => CarregarFuncionarios();

            // DataGridView
            dgvFuncionarios = CriarDataGridView();
            dgvFuncionarios.Location = new Point(20, 70);
            dgvFuncionarios.Size = new Size(1120, 500);

            tabFuncionarios.Controls.AddRange(new Control[] { 
                lblBuscaFuncionario, txtBuscaFuncionario, btnAtualizarFuncionarios, dgvFuncionarios 
            });
            tabControl.TabPages.Add(tabFuncionarios);
        }

        #endregion

        #region MÉTODOS AUXILIARES

        private DataGridView CriarDataGridView()
        {
            DataGridView dgv = new DataGridView
            {
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                RowHeadersVisible = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                Font = new Font("Segoe UI", 9)
            };

            dgv.EnableHeadersVisualStyles = false;
            dgv.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(52, 73, 94);
            dgv.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
            dgv.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 10, FontStyle.Bold);
            dgv.ColumnHeadersHeight = 35;

            return dgv;
        }

        #endregion

        #region CARREGAMENTO DE DADOS

        private void CarregarDados()
        {
            CarregarMaquinas();
            CarregarPecas();
            CarregarFuncionarios();
        }

        private void CarregarMaquinas()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_maquina AS ID,
                            planta AS Planta,
                            nome_maquina AS 'Nome da Máquina',
                            status_operacional AS Status,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y %H:%i') AS 'Data Cadastro'
                        FROM maquinas
                        ORDER BY data_cadastro DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvMaquinas.DataSource = dt;

                    if (dgvMaquinas.Columns["ID"] != null)
                        dgvMaquinas.Columns["ID"].Width = 60;

                    // Exibir total
                    tabMaquinas.Text = $"⚙️ Máquinas ({dt.Rows.Count})";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar máquinas: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CarregarPecas()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_peca AS ID,
                            nome_peca AS 'Nome da Peça',
                            codigo_peca AS Código,
                            estoque_atual AS 'Estoque Atual',
                            estoque_minimo AS 'Estoque Mínimo',
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y %H:%i') AS 'Data Cadastro'
                        FROM pecas
                        ORDER BY data_cadastro DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvPecas.DataSource = dt;

                    if (dgvPecas.Columns["ID"] != null)
                        dgvPecas.Columns["ID"].Width = 60;

                    // Exibir total
                    tabPecas.Text = $"🔧 Peças ({dt.Rows.Count})";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar peças: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CarregarFuncionarios()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_funcionario AS ID,
                            nome AS Nome,
                            telefone AS Telefone,
                            whatsapp AS WhatsApp,
                            cargo AS Cargo,
                            status AS Status,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y %H:%i') AS 'Data Cadastro'
                        FROM funcionarios
                        ORDER BY data_cadastro DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvFuncionarios.DataSource = dt;

                    if (dgvFuncionarios.Columns["ID"] != null)
                        dgvFuncionarios.Columns["ID"].Width = 60;

                    // Exibir total
                    tabFuncionarios.Text = $"👤 Funcionários ({dt.Rows.Count})";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar funcionários: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        #endregion

        #region FILTROS

        private void FiltrarMaquinas()
        {
            try
            {
                if (dgvMaquinas.DataSource is DataTable dt)
                {
                    string filtro = txtBuscaMaquina.Text.Trim();
                    if (string.IsNullOrEmpty(filtro))
                    {
                        dt.DefaultView.RowFilter = string.Empty;
                    }
                    else
                    {
                        dt.DefaultView.RowFilter = $"Planta LIKE '%{filtro}%' OR " +
                                                   $"[Nome da Máquina] LIKE '%{filtro}%' OR " +
                                                   $"Status LIKE '%{filtro}%'";
                    }
                }
            }
            catch { }
        }

        private void FiltrarPecas()
        {
            try
            {
                if (dgvPecas.DataSource is DataTable dt)
                {
                    string filtro = txtBuscaPeca.Text.Trim();
                    if (string.IsNullOrEmpty(filtro))
                    {
                        dt.DefaultView.RowFilter = string.Empty;
                    }
                    else
                    {
                        dt.DefaultView.RowFilter = $"[Nome da Peça] LIKE '%{filtro}%' OR " +
                                                   $"Código LIKE '%{filtro}%'";
                    }
                }
            }
            catch { }
        }

        private void FiltrarFuncionarios()
        {
            try
            {
                if (dgvFuncionarios.DataSource is DataTable dt)
                {
                    string filtro = txtBuscaFuncionario.Text.Trim();
                    if (string.IsNullOrEmpty(filtro))
                    {
                        dt.DefaultView.RowFilter = string.Empty;
                    }
                    else
                    {
                        dt.DefaultView.RowFilter = $"Nome LIKE '%{filtro}%' OR " +
                                                   $"Telefone LIKE '%{filtro}%' OR " +
                                                   $"WhatsApp LIKE '%{filtro}%' OR " +
                                                   $"Cargo LIKE '%{filtro}%' OR " +
                                                   $"Status LIKE '%{filtro}%'";
                    }
                }
            }
            catch { }
        }

        #endregion
    }
}

using System;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmEditarUsuario : Form
    {
        private TextBox txtNome, txtLogin, txtSenha, txtConfirmarSenha, txtTelefone, txtEmail;
        private ComboBox cmbTipo;
        private CheckBox chkAtivo;
        private Button btnSalvar, btnCancelar;
        private int? idUsuario;
        private bool modoEdicao;

        public FrmEditarUsuario(int? idUsuarioEditar = null)
        {
            idUsuario = idUsuarioEditar;
            modoEdicao = idUsuarioEditar.HasValue;

            ConfigurarFormulario();
            CriarInterface();

            if (modoEdicao)
            {
                CarregarDadosUsuario();
            }
        }

        private void ConfigurarFormulario()
        {
            this.Text = modoEdicao ? "Editar Usuário" : "Novo Usuário";
            this.Size = new Size(550, 550);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterParent;
            this.FormBorderStyle = FormBorderStyle.FixedDialog;
            this.MaximizeBox = false;
            this.MinimizeBox = false;
        }

        private void CriarInterface()
        {
            Label lblTitulo = new Label
            {
                Text = modoEdicao ? "✏️ Editar Usuário" : "➕ Novo Usuário",
                Font = new Font("Segoe UI", 16, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            int yPos = 80;

            // Nome
            Label lblNome = new Label
            {
                Text = "Nome:*",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtNome = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(320, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Login
            Label lblLogin = new Label
            {
                Text = "Login:*",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtLogin = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Senha
            Label lblSenha = new Label
            {
                Text = modoEdicao ? "Nova Senha:" : "Senha:*",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtSenha = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                PasswordChar = '●'
            };
            yPos += 45;

            // Confirmar Senha
            Label lblConfirmarSenha = new Label
            {
                Text = "Confirmar Senha:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtConfirmarSenha = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                PasswordChar = '●'
            };
            yPos += 45;

            // Tipo
            Label lblTipo = new Label
            {
                Text = "Tipo:*",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            cmbTipo = new ComboBox
            {
                Location = new Point(180, yPos),
                Size = new Size(150, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };
            cmbTipo.Items.AddRange(new object[] { "USUARIO", "ADMIN" });
            cmbTipo.SelectedIndex = 0;
            yPos += 45;

            // Telefone
            Label lblTelefone = new Label
            {
                Text = "Telefone:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtTelefone = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Email
            Label lblEmail = new Label
            {
                Text = "E-mail:",
                Location = new Point(50, yPos),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtEmail = new TextBox
            {
                Location = new Point(180, yPos),
                Size = new Size(320, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Ativo
            chkAtivo = new CheckBox
            {
                Text = "Usuário Ativo",
                Location = new Point(180, yPos),
                Size = new Size(150, 25),
                Font = new Font("Segoe UI", 10),
                Checked = true
            };

            // Botões
            btnSalvar = new Button
            {
                Text = "💾 Salvar",
                Location = new Point(300, 460),
                Size = new Size(100, 35),
                BackColor = Color.FromArgb(46, 204, 113),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnSalvar.FlatAppearance.BorderSize = 0;
            btnSalvar.Click += BtnSalvar_Click;

            btnCancelar = new Button
            {
                Text = "✖ Cancelar",
                Location = new Point(410, 460),
                Size = new Size(100, 35),
                BackColor = Color.FromArgb(149, 165, 166),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnCancelar.FlatAppearance.BorderSize = 0;
            btnCancelar.Click += (s, e) => this.Close();

            this.Controls.AddRange(new Control[] {
                lblTitulo,
                lblNome, txtNome,
                lblLogin, txtLogin,
                lblSenha, txtSenha,
                lblConfirmarSenha, txtConfirmarSenha,
                lblTipo, cmbTipo,
                lblTelefone, txtTelefone,
                lblEmail, txtEmail,
                chkAtivo,
                btnSalvar, btnCancelar
            });
        }

        private void CarregarDadosUsuario()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"SELECT nome, login, tipo_usuario, telefone, email, ativo 
                                    FROM usuarios WHERE id_usuario = @id";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@id", idUsuario);

                    MySqlDataReader reader = cmd.ExecuteReader();

                    if (reader.Read())
                    {
                        txtNome.Text = reader["nome"].ToString();
                        txtLogin.Text = reader["login"].ToString();
                        cmbTipo.SelectedItem = reader["tipo_usuario"].ToString();
                        txtTelefone.Text = reader["telefone"]?.ToString() ?? "";
                        txtEmail.Text = reader["email"]?.ToString() ?? "";
                        chkAtivo.Checked = Convert.ToBoolean(reader["ativo"]);
                    }

                    reader.Close();
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar dados: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            // Validações
            if (string.IsNullOrWhiteSpace(txtNome.Text))
            {
                MessageBox.Show("Informe o nome!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtNome.Focus();
                return;
            }

            if (string.IsNullOrWhiteSpace(txtLogin.Text))
            {
                MessageBox.Show("Informe o login!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtLogin.Focus();
                return;
            }

            // Validar senha apenas se for novo usuário ou se preencheu senha
            if (!modoEdicao || !string.IsNullOrEmpty(txtSenha.Text))
            {
                if (string.IsNullOrEmpty(txtSenha.Text))
                {
                    MessageBox.Show("Informe a senha!", "Validação",
                        MessageBoxButtons.OK, MessageBoxIcon.Warning);
                    txtSenha.Focus();
                    return;
                }

                if (txtSenha.Text.Length < 6)
                {
                    MessageBox.Show("A senha deve ter no mínimo 6 caracteres!", "Validação",
                        MessageBoxButtons.OK, MessageBoxIcon.Warning);
                    txtSenha.Focus();
                    return;
                }

                if (txtSenha.Text != txtConfirmarSenha.Text)
                {
                    MessageBox.Show("As senhas não coincidem!", "Validação",
                        MessageBoxButtons.OK, MessageBoxIcon.Warning);
                    txtConfirmarSenha.Focus();
                    return;
                }
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query;

                    if (modoEdicao)
                    {
                        // Atualizar
                        if (string.IsNullOrEmpty(txtSenha.Text))
                        {
                            query = @"UPDATE usuarios 
                                    SET nome = @nome, 
                                        login = @login, 
                                        tipo_usuario = @tipo,
                                        telefone = @telefone,
                                        email = @email,
                                        ativo = @ativo
                                    WHERE id_usuario = @id";
                        }
                        else
                        {
                            query = @"UPDATE usuarios 
                                    SET nome = @nome, 
                                        login = @login, 
                                        senha = @senha,
                                        tipo_usuario = @tipo,
                                        telefone = @telefone,
                                        email = @email,
                                        ativo = @ativo
                                    WHERE id_usuario = @id";
                        }
                    }
                    else
                    {
                        // Inserir
                        query = @"INSERT INTO usuarios 
                                (nome, login, senha, tipo_usuario, telefone, email, ativo)
                                VALUES (@nome, @login, @senha, @tipo, @telefone, @email, @ativo)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@nome", txtNome.Text.Trim());
                    cmd.Parameters.AddWithValue("@login", txtLogin.Text.Trim());
                    
                    if (!modoEdicao || !string.IsNullOrEmpty(txtSenha.Text))
                    {
                        cmd.Parameters.AddWithValue("@senha", txtSenha.Text);
                    }
                    
                    cmd.Parameters.AddWithValue("@tipo", cmbTipo.SelectedItem.ToString());
                    cmd.Parameters.AddWithValue("@telefone", txtTelefone.Text.Trim());
                    cmd.Parameters.AddWithValue("@email", txtEmail.Text.Trim());
                    cmd.Parameters.AddWithValue("@ativo", chkAtivo.Checked);

                    if (modoEdicao)
                    {
                        cmd.Parameters.AddWithValue("@id", idUsuario);
                    }

                    cmd.ExecuteNonQuery();

                    MessageBox.Show(
                        modoEdicao ? "Usuário atualizado com sucesso!" : "Usuário criado com sucesso!",
                        "Sucesso",
                        MessageBoxButtons.OK,
                        MessageBoxIcon.Information
                    );

                    this.DialogResult = DialogResult.OK;
                    this.Close();
                }
            }
            catch (MySqlException ex) when (ex.Number == 1062) // Duplicate entry
            {
                MessageBox.Show("Este login já está sendo usado por outro usuário!", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}

using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmGerenciarUsuarios : Form
    {
        private DataGridView dgvUsuarios;
        private Button btnNovo, btnEditar, btnExcluir, btnAtualizar;
        private TextBox txtBuscar;

        public FrmGerenciarUsuarios()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarUsuarios();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Gerenciar Usuários - Admin";
            this.Size = new Size(1000, 600);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            Label lblTitulo = new Label
            {
                Text = "👥 Gerenciamento de Usuários",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // Buscar
            Label lblBuscar = new Label
            {
                Text = "🔍 Buscar:",
                Location = new Point(30, 80),
                Size = new Size(80, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtBuscar = new TextBox
            {
                Location = new Point(120, 80),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };
            txtBuscar.TextChanged += (s, e) => FiltrarUsuarios();

            // DataGridView
            dgvUsuarios = new DataGridView
            {
                Location = new Point(30, 130),
                Size = new Size(920, 350),
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                RowHeadersVisible = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                Font = new Font("Segoe UI", 9)
            };

            dgvUsuarios.EnableHeadersVisualStyles = false;
            dgvUsuarios.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(52, 73, 94);
            dgvUsuarios.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
            dgvUsuarios.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 10, FontStyle.Bold);
            dgvUsuarios.ColumnHeadersHeight = 35;

            // Botões
            btnNovo = new Button
            {
                Text = "➕ Novo Usuário",
                Location = new Point(30, 500),
                Size = new Size(150, 40),
                BackColor = Color.FromArgb(46, 204, 113),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnNovo.FlatAppearance.BorderSize = 0;
            btnNovo.Click += BtnNovo_Click;

            btnEditar = new Button
            {
                Text = "✏️ Editar",
                Location = new Point(190, 500),
                Size = new Size(120, 40),
                BackColor = Color.FromArgb(230, 126, 34),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand,
                Enabled = false
            };
            btnEditar.FlatAppearance.BorderSize = 0;
            btnEditar.Click += BtnEditar_Click;

            btnExcluir = new Button
            {
                Text = "🗑️ Excluir",
                Location = new Point(320, 500),
                Size = new Size(120, 40),
                BackColor = Color.FromArgb(231, 76, 60),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand,
                Enabled = false
            };
            btnExcluir.FlatAppearance.BorderSize = 0;
            btnExcluir.Click += BtnExcluir_Click;

            btnAtualizar = new Button
            {
                Text = "🔄 Atualizar",
                Location = new Point(830, 500),
                Size = new Size(120, 40),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAtualizar.FlatAppearance.BorderSize = 0;
            btnAtualizar.Click += (s, e) => CarregarUsuarios();

            dgvUsuarios.SelectionChanged += (s, e) =>
            {
                bool temSelecao = dgvUsuarios.SelectedRows.Count > 0;
                btnEditar.Enabled = temSelecao;
                btnExcluir.Enabled = temSelecao;
            };

            this.Controls.AddRange(new Control[] {
                lblTitulo, lblBuscar, txtBuscar,
                dgvUsuarios, btnNovo, btnEditar, btnExcluir, btnAtualizar
            });
        }

        private void CarregarUsuarios()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_usuario AS ID,
                            nome AS Nome,
                            login AS Login,
                            tipo_usuario AS Tipo,
                            telefone AS Telefone,
                            email AS Email,
                            CASE WHEN ativo = 1 THEN 'Ativo' ELSE 'Inativo' END AS Status,
                            DATE_FORMAT(data_criacao, '%d/%m/%Y %H:%i') AS 'Cadastrado em',
                            DATE_FORMAT(data_ultimo_acesso, '%d/%m/%Y %H:%i') AS 'Último Acesso'
                        FROM usuarios
                        ORDER BY data_criacao DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvUsuarios.DataSource = dt;

                    if (dgvUsuarios.Columns["ID"] != null)
                        dgvUsuarios.Columns["ID"].Width = 50;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar usuários: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void FiltrarUsuarios()
        {
            try
            {
                if (dgvUsuarios.DataSource is DataTable dt)
                {
                    string filtro = txtBuscar.Text.Trim();
                    if (string.IsNullOrEmpty(filtro))
                    {
                        dt.DefaultView.RowFilter = string.Empty;
                    }
                    else
                    {
                        dt.DefaultView.RowFilter = $"Nome LIKE '%{filtro}%' OR " +
                                                   $"Login LIKE '%{filtro}%' OR " +
                                                   $"Tipo LIKE '%{filtro}%'";
                    }
                }
            }
            catch { }
        }

        private void InitializeComponent()
        {
            this.SuspendLayout();
            // 
            // FrmGerenciarUsuarios
            // 
            this.ClientSize = new System.Drawing.Size(284, 261);
            this.Name = "FrmGerenciarUsuarios";
            this.Load += new System.EventHandler(this.FrmGerenciarUsuarios_Load);
            this.ResumeLayout(false);

        }

        private void FrmGerenciarUsuarios_Load(object sender, EventArgs e)
        {

        }

        private void BtnNovo_Click(object sender, EventArgs e)
        {
            FrmEditarUsuario frm = new FrmEditarUsuario();
            if (frm.ShowDialog() == DialogResult.OK)
            {
                CarregarUsuarios();
            }
        }

        private void BtnEditar_Click(object sender, EventArgs e)
        {
            if (dgvUsuarios.SelectedRows.Count == 0) return;

            int idUsuario = Convert.ToInt32(dgvUsuarios.SelectedRows[0].Cells["ID"].Value);
            FrmEditarUsuario frm = new FrmEditarUsuario(idUsuario);
            if (frm.ShowDialog() == DialogResult.OK)
            {
                CarregarUsuarios();
            }
        }

        private void BtnExcluir_Click(object sender, EventArgs e)
        {
            if (dgvUsuarios.SelectedRows.Count == 0) return;

            int idUsuario = Convert.ToInt32(dgvUsuarios.SelectedRows[0].Cells["ID"].Value);
            string nomeUsuario = dgvUsuarios.SelectedRows[0].Cells["Nome"].Value.ToString();

            if (idUsuario == SessaoUsuario.IdUsuario)
            {
                MessageBox.Show("Você não pode excluir seu próprio usuário!", "Aviso",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            DialogResult result = MessageBox.Show(
                $"Deseja realmente excluir o usuário '{nomeUsuario}'?\n\nEsta ação não pode ser desfeita!",
                "Confirmar Exclusão",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                try
                {
                    using (MySqlConnection conn = ConexaoDB.ObterConexao())
                    {
                        string query = "DELETE FROM usuarios WHERE id_usuario = @id";
                        MySqlCommand cmd = new MySqlCommand(query, conn);
                        cmd.Parameters.AddWithValue("@id", idUsuario);
                        cmd.ExecuteNonQuery();

                        MessageBox.Show("Usuário excluído com sucesso!", "Sucesso",
                            MessageBoxButtons.OK, MessageBoxIcon.Information);

                        CarregarUsuarios();
                    }
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao excluir: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }
    }
}

using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmHistorico : Form
    {
        // Controles
        private Panel pnlFiltros;
        private Label lblTitulo;
        private ComboBox cmbStatus, cmbPlanta, cmbPeca, cmbMaquina;
        private Button btnFiltrar, btnLimpar;
        private DataGridView dgvHistorico;
        private Label lblTotal;

        public FrmHistorico()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarCombos();
            CarregarHistorico();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Histórico de Manutenções";
            this.Size = new Size(1400, 800);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // Título
            lblTitulo = new Label
            {
                Text = "📋 Histórico de Manutenções",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // Painel de Filtros
            pnlFiltros = new Panel
            {
                Location = new Point(30, 80),
                Size = new Size(1320, 100),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // Status
            Label lblStatus = new Label
            {
                Text = "Status:",
                Location = new Point(20, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbStatus = new ComboBox
            {
                Location = new Point(20, 45),
                Size = new Size(200, 30),
                DropDownStyle = ComboBoxStyle.DropDownList,
                Font = new Font("Segoe UI", 10)
            };
            cmbStatus.Items.AddRange(new object[] { "TODOS", "REALIZADO", "PLANEJADO", "ATRASADO", "HOJE" });
            cmbStatus.SelectedIndex = 0;

            // Planta
            Label lblPlanta = new Label
            {
                Text = "Planta:",
                Location = new Point(240, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbPlanta = new ComboBox
            {
                Location = new Point(240, 45),
                Size = new Size(200, 30),
                DropDownStyle = ComboBoxStyle.DropDownList,
                Font = new Font("Segoe UI", 10)
            };

            // Máquina
            Label lblMaquina = new Label
            {
                Text = "Máquina:",
                Location = new Point(460, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbMaquina = new ComboBox
            {
                Location = new Point(460, 45),
                Size = new Size(300, 30),
                DropDownStyle = ComboBoxStyle.DropDownList,
                Font = new Font("Segoe UI", 10)
            };

            // Peça
            Label lblPeca = new Label
            {
                Text = "Peça:",
                Location = new Point(780, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbPeca = new ComboBox
            {
                Location = new Point(780, 45),
                Size = new Size(300, 30),
                DropDownStyle = ComboBoxStyle.DropDownList,
                Font = new Font("Segoe UI", 10)
            };

            // Botões
            btnFiltrar = new Button
            {
                Text = "🔍 Filtrar",
                Location = new Point(1100, 35),
                Size = new Size(100, 40),
                BackColor = Color.FromArgb(0, 123, 255),
                ForeColor = Color.White,
                FlatStyle = FlatStyle.Flat,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                Cursor = Cursors.Hand
            };
            btnFiltrar.FlatAppearance.BorderSize = 0;
            btnFiltrar.Click += BtnFiltrar_Click;

            btnLimpar = new Button
            {
                Text = "🗑️ Limpar",
                Location = new Point(1210, 35),
                Size = new Size(100, 40),
                BackColor = Color.FromArgb(108, 117, 125),
                ForeColor = Color.White,
                FlatStyle = FlatStyle.Flat,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                Cursor = Cursors.Hand
            };
            btnLimpar.FlatAppearance.BorderSize = 0;
            btnLimpar.Click += BtnLimpar_Click;

            pnlFiltros.Controls.AddRange(new Control[] {
                lblStatus, cmbStatus,
                lblPlanta, cmbPlanta,
                lblMaquina, cmbMaquina,
                lblPeca, cmbPeca,
                btnFiltrar, btnLimpar
            });

            // DataGridView
            dgvHistorico = new DataGridView
            {
                Location = new Point(30, 200),
                Size = new Size(1320, 490),
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                RowHeadersVisible = false,
                Font = new Font("Segoe UI", 9),
                ColumnHeadersDefaultCellStyle = new DataGridViewCellStyle
                {
                    BackColor = Color.FromArgb(52, 58, 64),
                    ForeColor = Color.White,
                    Font = new Font("Segoe UI", 10, FontStyle.Bold),
                    Alignment = DataGridViewContentAlignment.MiddleCenter
                },
                DefaultCellStyle = new DataGridViewCellStyle
                {
                    SelectionBackColor = Color.FromArgb(0, 123, 255),
                    SelectionForeColor = Color.White
                },
                EnableHeadersVisualStyles = false
            };

            // Adicionar evento de duplo clique
            dgvHistorico.CellDoubleClick += DgvHistorico_CellDoubleClick;

            // Label Total
            lblTotal = new Label
            {
                Text = "Total: 0 manutenções",
                Location = new Point(30, 700),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64)
            };

            // Adicionar ao formulário
            this.Controls.AddRange(new Control[] {
                lblTitulo,
                pnlFiltros,
                dgvHistorico,
                lblTotal
            });
        }

        private void CarregarCombos()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // Plantas
                    if (cmbPlanta != null)
                    {
                        cmbPlanta.Items.Clear();
                        cmbPlanta.Items.Add("TODAS");
                        
                        string queryPlantas = "SELECT DISTINCT planta FROM maquinas ORDER BY planta";
                        MySqlCommand cmdPlantas = new MySqlCommand(queryPlantas, conn);
                        MySqlDataReader readerPlantas = cmdPlantas.ExecuteReader();
                        while (readerPlantas.Read())
                        {
                            string planta = readerPlantas["planta"]?.ToString();
                            if (!string.IsNullOrEmpty(planta))
                                cmbPlanta.Items.Add(planta);
                        }
                        readerPlantas.Close();
                        cmbPlanta.SelectedIndex = 0;
                    }

                    // Máquinas
                    if (cmbMaquina != null)
                    {
                        cmbMaquina.Items.Clear();
                        cmbMaquina.Items.Add("TODAS");
                        
                        string queryMaquinas = "SELECT DISTINCT nome_maquina FROM maquinas ORDER BY nome_maquina";
                        MySqlCommand cmdMaquinas = new MySqlCommand(queryMaquinas, conn);
                        MySqlDataReader readerMaquinas = cmdMaquinas.ExecuteReader();
                        while (readerMaquinas.Read())
                        {
                            string maquina = readerMaquinas["nome_maquina"]?.ToString();
                            if (!string.IsNullOrEmpty(maquina))
                                cmbMaquina.Items.Add(maquina);
                        }
                        readerMaquinas.Close();
                        cmbMaquina.SelectedIndex = 0;
                    }

                    // Peças
                    if (cmbPeca != null)
                    {
                        cmbPeca.Items.Clear();
                        cmbPeca.Items.Add("TODAS");
                        
                        string queryPecas = "SELECT DISTINCT nome_peca FROM pecas ORDER BY nome_peca";
                        MySqlCommand cmdPecas = new MySqlCommand(queryPecas, conn);
                        MySqlDataReader readerPecas = cmdPecas.ExecuteReader();
                        while (readerPecas.Read())
                        {
                            string peca = readerPecas["nome_peca"]?.ToString();
                            if (!string.IsNullOrEmpty(peca))
                                cmbPeca.Items.Add(peca);
                        }
                        readerPecas.Close();
                        cmbPeca.SelectedIndex = 0;
                    }
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar filtros: {ex.Message}\n\nDetalhes: {ex.StackTrace}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void CarregarHistorico()
        {
            try
            {
                if (dgvHistorico == null)
                {
                    MessageBox.Show("DataGridView não inicializado!", "Erro", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    return;
                }

                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            m.id_manutencao AS ID,
                            m.planta AS Planta,
                            maq.nome_maquina AS Máquina,
                            p.nome_peca AS Peça,
                            m.quantidade AS Qtd,
                            DATE_FORMAT(m.data_planejada, '%d/%m/%Y') AS 'Data Planejada',
                            DATE_FORMAT(m.data_realizada, '%d/%m/%Y') AS 'Data Realizada',
                            m.situacao AS Status,
                            m.horas_trabalhadas AS 'Horas Trab.',
                            m.horas_previstas AS 'Horas Prev.',
                            COALESCE(f.nome, '-') AS Funcionário,
                            COALESCE(m.observacoes, '') AS Observações
                        FROM manutencoes m
                        INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
                        INNER JOIN pecas p ON m.id_peca = p.id_peca
                        LEFT JOIN funcionarios f ON m.id_funcionario = f.id_funcionario
                        WHERE 1=1";

                    // Aplicar filtros
                    if (cmbStatus != null && cmbStatus.SelectedItem != null && cmbStatus.SelectedItem.ToString() != "TODOS")
                    {
                        query += " AND m.situacao = @status";
                    }

                    if (cmbPlanta != null && cmbPlanta.SelectedItem != null && cmbPlanta.SelectedItem.ToString() != "TODAS")
                    {
                        query += " AND m.planta = @planta";
                    }

                    if (cmbMaquina != null && cmbMaquina.SelectedItem != null && cmbMaquina.SelectedItem.ToString() != "TODAS")
                    {
                        query += " AND maq.nome_maquina = @maquina";
                    }

                    if (cmbPeca != null && cmbPeca.SelectedItem != null && cmbPeca.SelectedItem.ToString() != "TODAS")
                    {
                        query += " AND p.nome_peca = @peca";
                    }

                    query += " ORDER BY m.data_planejada DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);

                    // Adicionar parâmetros
                    if (cmbStatus != null && cmbStatus.SelectedItem != null && cmbStatus.SelectedItem.ToString() != "TODOS")
                        cmd.Parameters.AddWithValue("@status", cmbStatus.SelectedItem.ToString());

                    if (cmbPlanta != null && cmbPlanta.SelectedItem != null && cmbPlanta.SelectedItem.ToString() != "TODAS")
                        cmd.Parameters.AddWithValue("@planta", cmbPlanta.SelectedItem.ToString());

                    if (cmbMaquina != null && cmbMaquina.SelectedItem != null && cmbMaquina.SelectedItem.ToString() != "TODAS")
                        cmd.Parameters.AddWithValue("@maquina", cmbMaquina.SelectedItem.ToString());

                    if (cmbPeca != null && cmbPeca.SelectedItem != null && cmbPeca.SelectedItem.ToString() != "TODAS")
                        cmd.Parameters.AddWithValue("@peca", cmbPeca.SelectedItem.ToString());

                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvHistorico.DataSource = dt;

                    // Configurar larguras das colunas
                    if (dgvHistorico.Columns.Contains("ID"))
                    {
                        dgvHistorico.Columns["ID"].AutoSizeMode = DataGridViewAutoSizeColumnMode.None;
                        dgvHistorico.Columns["ID"].Width = 60;
                    }

                    // Colorir linhas por status
                    foreach (DataGridViewRow row in dgvHistorico.Rows)
                    {
                        if (row.Cells["Status"] == null || row.Cells["Status"].Value == null)
                            continue;

                        string status = row.Cells["Status"].Value.ToString();

                        switch (status)
                        {
                            case "ATRASADO":
                                row.DefaultCellStyle.BackColor = Color.FromArgb(255, 230, 230);
                                break;
                            case "HOJE":
                                row.DefaultCellStyle.BackColor = Color.FromArgb(255, 250, 205);
                                break;
                            case "PLANEJADO":
                                row.DefaultCellStyle.BackColor = Color.FromArgb(230, 240, 255);
                                break;
                            case "REALIZADO":
                                row.DefaultCellStyle.BackColor = Color.FromArgb(230, 255, 230);
                                break;
                        }
                    }

                    if (lblTotal != null)
                        lblTotal.Text = $"Total: {dt.Rows.Count} manutenções";
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar histórico: {ex.Message}\n\nDetalhes: {ex.StackTrace}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        // ÚNICO método CellDoubleClick - Abre formulário de edição
        private void DgvHistorico_CellDoubleClick(object sender, DataGridViewCellEventArgs e)
        {
            try
            {
                if (e.RowIndex < 0 || dgvHistorico == null || dgvHistorico.Rows.Count == 0)
                    return;

                if (e.RowIndex >= dgvHistorico.Rows.Count)
                    return;

                DataGridViewRow row = dgvHistorico.Rows[e.RowIndex];

                if (row == null || row.Cells == null || row.Cells["ID"] == null || row.Cells["ID"].Value == null)
                    return;

                // Obter ID da manutenção
                int idManutencao = Convert.ToInt32(row.Cells["ID"].Value);

                // Abrir formulário de manutenções já com o registro carregado para edição
                FrmCadastroManutencao frmManutencao = new FrmCadastroManutencao(idManutencao);
                frmManutencao.Show();
                
                // Fechar histórico
                this.Close();
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao abrir manutenção: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnFiltrar_Click(object sender, EventArgs e)
        {
            CarregarHistorico();
        }

        private void BtnLimpar_Click(object sender, EventArgs e)
        {
            cmbStatus.SelectedIndex = 0;
            cmbPlanta.SelectedIndex = 0;
            cmbMaquina.SelectedIndex = 0;
            cmbPeca.SelectedIndex = 0;
            CarregarHistorico();
        }
    }
}

using System;
using System.Drawing;
using System.IO;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmLogin : Form
    {
        private TextBox txtLogin, txtSenha;
        private Button btnEntrar, btnPularLogin;
        private CheckBox chkLembrar;
        private Label lblTitulo, lblSubtitulo, lblErro;
        private Panel pnlLogin;

        public bool LoginCancelado { get; private set; }

        public FrmLogin()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarCredenciaisSalvas();
            
            // Garantir que ao fechar com X, retorna Cancel
            this.FormClosing += FrmLogin_FormClosing;
        }

        private void FrmLogin_FormClosing(object sender, FormClosingEventArgs e)
        {
            // Se não definiu DialogResult (fechou com X), definir como Cancel
            if (this.DialogResult != DialogResult.OK && this.DialogResult != DialogResult.Cancel)
            {
                this.DialogResult = DialogResult.Cancel;
            }
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Login - Sistema de Manutenção";
            this.Size = new Size(500, 600);
            this.BackColor = Color.FromArgb(44, 62, 80);
            this.StartPosition = FormStartPosition.CenterScreen;
            this.FormBorderStyle = FormBorderStyle.FixedDialog;
            this.MaximizeBox = false;
            this.MinimizeBox = false;
        }

        private void CriarInterface()
        {
            // Painel central
            pnlLogin = new Panel
            {
                Size = new Size(400, 450),
                Location = new Point(50, 75),
                BackColor = Color.White
            };

            // Ícone
            Label lblIcone = new Label
            {
                Text = "🔧",
                Font = new Font("Segoe UI", 48),
                ForeColor = Color.FromArgb(52, 152, 219),
                TextAlign = ContentAlignment.MiddleCenter,
                Location = new Point(0, 30),
                Size = new Size(400, 80)
            };

            // Título
            lblTitulo = new Label
            {
                Text = "SISTEMA DE MANUTENÇÃO",
                Font = new Font("Segoe UI", 16, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                TextAlign = ContentAlignment.MiddleCenter,
                Location = new Point(0, 120),
                Size = new Size(400, 30)
            };

            lblSubtitulo = new Label
            {
                Text = "Faça login para continuar",
                Font = new Font("Segoe UI", 10),
                ForeColor = Color.FromArgb(127, 140, 141),
                TextAlign = ContentAlignment.MiddleCenter,
                Location = new Point(0, 155),
                Size = new Size(400, 25)
            };

            // Label Login
            Label lblLoginText = new Label
            {
                Text = "Usuário:",
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(50, 200),
                Size = new Size(80, 25)
            };

            // Campo Login
            txtLogin = new TextBox
            {
                Location = new Point(50, 230),
                Size = new Size(300, 35),
                Font = new Font("Segoe UI", 12),
                Text = ""
            };

            // Label Senha
            Label lblSenhaText = new Label
            {
                Text = "Senha:",
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(50, 275),
                Size = new Size(80, 25)
            };

            // Campo Senha
            txtSenha = new TextBox
            {
                Location = new Point(50, 305),
                Size = new Size(300, 35),
                Font = new Font("Segoe UI", 12),
                PasswordChar = '●',
                Text = ""
            };
            txtSenha.KeyPress += (s, e) =>
            {
                if (e.KeyChar == (char)Keys.Enter)
                {
                    BtnEntrar_Click(null, null);
                    e.Handled = true;
                }
            };

            // Checkbox Lembrar
            chkLembrar = new CheckBox
            {
                Text = "Lembrar meu login",
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.FromArgb(127, 140, 141),
                Location = new Point(50, 350),
                Size = new Size(150, 25)
            };

            // Label de Erro
            lblErro = new Label
            {
                Text = "",
                Font = new Font("Segoe UI", 9, FontStyle.Bold),
                ForeColor = Color.FromArgb(231, 76, 60),
                TextAlign = ContentAlignment.MiddleCenter,
                Location = new Point(50, 380),
                Size = new Size(300, 25),
                Visible = false
            };

            // Botão Entrar
            btnEntrar = new Button
            {
                Text = "ENTRAR",
                Location = new Point(50, 415),
                Size = new Size(145, 45),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnEntrar.FlatAppearance.BorderSize = 0;
            btnEntrar.Click += BtnEntrar_Click;

            // Botão Pular Login
            btnPularLogin = new Button
            {
                Text = "PULAR LOGIN",
                Location = new Point(205, 415),
                Size = new Size(145, 45),
                BackColor = Color.FromArgb(149, 165, 166),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnPularLogin.FlatAppearance.BorderSize = 0;
            btnPularLogin.Click += BtnPularLogin_Click;

            // Label Informação
            Label lblInfo = new Label
            {
                Text = "Admin: admin / admin123  |  User: user / user123",
                Font = new Font("Segoe UI", 8),
                ForeColor = Color.FromArgb(149, 165, 166),
                TextAlign = ContentAlignment.MiddleCenter,
                Location = new Point(0, 470),
                Size = new Size(400, 20)
            };

            pnlLogin.Controls.AddRange(new Control[] {
                lblIcone, lblTitulo, lblSubtitulo,
                lblLoginText, txtLogin,
                lblSenhaText, txtSenha,
                chkLembrar, lblErro,
                btnEntrar, btnPularLogin, lblInfo
            });

            this.Controls.Add(pnlLogin);
        }

        private void BtnEntrar_Click(object sender, EventArgs e)
        {
            string login = txtLogin.Text.Trim();
            string senha = txtSenha.Text;

            if (string.IsNullOrEmpty(login) || string.IsNullOrEmpty(senha))
            {
                MostrarErro("Preencha todos os campos!");
                return;
            }

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"SELECT id_usuario, nome, login, senha, tipo_usuario, telefone, email, foto_perfil, ativo
                                    FROM usuarios 
                                    WHERE login = @login AND ativo = TRUE";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@login", login);

                    MySqlDataReader reader = cmd.ExecuteReader();

                    if (reader.Read())
                    {
                        string senhaArmazenada = reader["senha"].ToString();

                        // Verificar senha (em produção use hash!)
                        if (senha == senhaArmazenada)
                        {
                            int idUsuario = Convert.ToInt32(reader["id_usuario"]);
                            string nome = reader["nome"].ToString();
                            string tipo = reader["tipo_usuario"].ToString();
                            string telefone = reader["telefone"]?.ToString();
                            string email = reader["email"]?.ToString();
                            byte[] foto = reader["foto_perfil"] != DBNull.Value ? (byte[])reader["foto_perfil"] : null;

                            reader.Close();

                            // Iniciar sessão
                            SessaoUsuario.Iniciar(idUsuario, nome, login, tipo, telefone, email, foto);

                            // Salvar login se marcado
                            if (chkLembrar.Checked)
                            {
                                SalvarLoginLembrado(login);
                            }
                            else
                            {
                                SalvarLoginLembrado("");
                            }

                            this.DialogResult = DialogResult.OK;
                            this.Close();
                        }
                        else
                        {
                            reader.Close();
                            RegistrarTentativaFalha(login);
                            MostrarErro("Senha incorreta!");
                        }
                    }
                    else
                    {
                        reader.Close();
                        MostrarErro("Usuário não encontrado!");
                    }
                }
            }
            catch (Exception ex)
            {
                MostrarErro($"Erro: {ex.Message}");
            }
        }

        private void BtnPularLogin_Click(object sender, EventArgs e)
        {
            DialogResult result = MessageBox.Show(
                "⚠️ MODO SEM LOGIN\n\n" +
                "Você está entrando sem fazer login.\n\n" +
                "Funções limitadas:\n" +
                "• Não pode alterar cadastros\n" +
                "• Não pode realizar manutenções\n" +
                "• Apenas visualização de dados\n\n" +
                "O sistema continuará enviando notificações automáticas.\n\n" +
                "Deseja continuar?",
                "Confirmar",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                LoginCancelado = true;
                this.DialogResult = DialogResult.Cancel;
                this.Close();
            }
        }

        private void MostrarErro(string mensagem)
        {
            lblErro.Text = mensagem;
            lblErro.Visible = true;

            Timer timer = new Timer { Interval = 3000 };
            timer.Tick += (s, e) =>
            {
                lblErro.Visible = false;
                timer.Stop();
                timer.Dispose();
            };
            timer.Start();
        }

        private void RegistrarTentativaFalha(string login)
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = "INSERT INTO log_acessos (id_usuario, tipo_acao) VALUES (NULL, 'ACESSO_NEGADO')";
                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.ExecuteNonQuery();
                }
            }
            catch { }
        }

        private void CarregarCredenciaisSalvas()
        {
            try
            {
                string caminhoArquivo = Path.Combine(Application.StartupPath, "login_salvo.txt");
                if (File.Exists(caminhoArquivo))
                {
                    string loginSalvo = File.ReadAllText(caminhoArquivo).Trim();
                    if (!string.IsNullOrEmpty(loginSalvo))
                    {
                        txtLogin.Text = loginSalvo;
                        chkLembrar.Checked = true;
                        txtSenha.Focus();
                        return;
                    }
                }
            }
            catch { }
            
            txtLogin.Focus();
        }

        private void SalvarLoginLembrado(string login)
        {
            try
            {
                string caminhoArquivo = Path.Combine(Application.StartupPath, "login_salvo.txt");
                File.WriteAllText(caminhoArquivo, login);
            }
            catch { }
        }
    }
}

using System;
using System.Data;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmMaquinas : Form
    {
        // Controles de Formulário
        private Panel pnlCadastro;
        private Label lblTitulo;
        private TextBox txtId, txtNomeMaquina, txtLocalizacao;
        private ComboBox cmbPlanta, cmbStatus;
        private Button btnNovo, btnSalvar, btnEditar, btnExcluir, btnCancelar;
        
        // Grid
        private DataGridView dgvMaquinas;
        
        // Controle
        private int idSelecionado = 0;
        private bool modoEdicao = false;

        public FrmMaquinas()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarMaquinas();
            LimparCampos();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Cadastro de Máquinas";
            this.Size = new Size(1200, 700);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // Título
            lblTitulo = new Label
            {
                Text = "⚙️ Cadastro de Máquinas",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // Painel de Cadastro
            pnlCadastro = new Panel
            {
                Location = new Point(30, 80),
                Size = new Size(1120, 200),
                BackColor = Color.FromArgb(248, 249, 250),
                BorderStyle = BorderStyle.FixedSingle
            };

            // ID
            Label lblId = new Label
            {
                Text = "ID:",
                Location = new Point(20, 20),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtId = new TextBox
            {
                Location = new Point(130, 20),
                Size = new Size(100, 30),
                Enabled = false,
                BackColor = Color.LightGray,
                Font = new Font("Segoe UI", 10)
            };

            // Planta
            Label lblPlanta = new Label
            {
                Text = "Planta:*",
                Location = new Point(20, 60),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbPlanta = new ComboBox
            {
                Location = new Point(130, 60),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };
            cmbPlanta.Items.AddRange(new object[] { "ITENSÃO", "REHTOM", "RIANOV" });

            // Nome da Máquina
            Label lblNome = new Label
            {
                Text = "Nome Máquina:*",
                Location = new Point(350, 60),
                Size = new Size(120, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtNomeMaquina = new TextBox
            {
                Location = new Point(480, 60),
                Size = new Size(400, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Localização
            Label lblLocalizacao = new Label
            {
                Text = "Localização:",
                Location = new Point(20, 100),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            txtLocalizacao = new TextBox
            {
                Location = new Point(130, 100),
                Size = new Size(300, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Status
            Label lblStatus = new Label
            {
                Text = "Status:*",
                Location = new Point(450, 100),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            cmbStatus = new ComboBox
            {
                Location = new Point(560, 100),
                Size = new Size(150, 30),
                Font = new Font("Segoe UI", 10),
                DropDownStyle = ComboBoxStyle.DropDownList
            };
            cmbStatus.Items.AddRange(new object[] { "ativo", "inativo", "manutencao" });
            cmbStatus.SelectedIndex = 0;

            // Botões
            int btnY = 145;
            int btnX = 20;

            btnNovo = CriarBotao("➕ Novo", btnX, btnY, Color.FromArgb(0, 123, 255));
            btnNovo.Click += BtnNovo_Click;
            btnX += 120;

            btnSalvar = CriarBotao("💾 Salvar", btnX, btnY, Color.FromArgb(40, 167, 69));
            btnSalvar.Click += BtnSalvar_Click;
            btnX += 120;

            btnEditar = CriarBotao("✏️ Editar", btnX, btnY, Color.FromArgb(255, 193, 7));
            btnEditar.Click += BtnEditar_Click;
            btnX += 120;

            btnExcluir = CriarBotao("🗑️ Excluir", btnX, btnY, Color.FromArgb(220, 53, 69));
            btnExcluir.Click += BtnExcluir_Click;
            btnX += 120;

            btnCancelar = CriarBotao("✖ Cancelar", btnX, btnY, Color.FromArgb(108, 117, 125));
            btnCancelar.Click += BtnCancelar_Click;

            pnlCadastro.Controls.AddRange(new Control[] {
                lblId, txtId,
                lblPlanta, cmbPlanta,
                lblNome, txtNomeMaquina,
                lblLocalizacao, txtLocalizacao,
                lblStatus, cmbStatus,
                btnNovo, btnSalvar, btnEditar, btnExcluir, btnCancelar
            });

            // DataGridView
            dgvMaquinas = new DataGridView
            {
                Location = new Point(30, 300),
                Size = new Size(1120, 350),
                BackgroundColor = Color.White,
                BorderStyle = BorderStyle.FixedSingle,
                AllowUserToAddRows = false,
                AllowUserToDeleteRows = false,
                ReadOnly = true,
                SelectionMode = DataGridViewSelectionMode.FullRowSelect,
                MultiSelect = false,
                RowHeadersVisible = false,
                AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill,
                Font = new Font("Segoe UI", 9)
            };

            dgvMaquinas.SelectionChanged += DgvMaquinas_SelectionChanged;

            // Estilo do cabeçalho
            dgvMaquinas.EnableHeadersVisualStyles = false;
            dgvMaquinas.ColumnHeadersDefaultCellStyle.BackColor = Color.FromArgb(52, 73, 94);
            dgvMaquinas.ColumnHeadersDefaultCellStyle.ForeColor = Color.White;
            dgvMaquinas.ColumnHeadersDefaultCellStyle.Font = new Font("Segoe UI", 10, FontStyle.Bold);
            dgvMaquinas.ColumnHeadersHeight = 35;

            // Adicionar controles ao formulário
            this.Controls.AddRange(new Control[] { lblTitulo, pnlCadastro, dgvMaquinas });
        }

        private Button CriarBotao(string texto, int x, int y, Color cor)
        {
            Button btn = new Button
            {
                Text = texto,
                Location = new Point(x, y),
                Size = new Size(110, 40),
                BackColor = cor,
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btn.FlatAppearance.BorderSize = 0;
            return btn;
        }

        private void CarregarMaquinas()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            id_maquina AS ID,
                            planta AS Planta,
                            nome_maquina AS 'Nome Máquina',
                            localizacao AS 'Localização',
                            status_operacional AS Status,
                            DATE_FORMAT(data_cadastro, '%d/%m/%Y') AS 'Data Cadastro'
                        FROM maquinas
                        ORDER BY id_maquina DESC";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataAdapter adapter = new MySqlDataAdapter(cmd);
                    DataTable dt = new DataTable();
                    adapter.Fill(dt);

                    dgvMaquinas.DataSource = dt;

                    // Ajustar largura das colunas
                    if (dgvMaquinas.Columns["ID"] != null)
                        dgvMaquinas.Columns["ID"].Width = 60;
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao carregar máquinas: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnNovo_Click(object sender, EventArgs e)
        {
            LimparCampos();
            HabilitarCampos(true);
            modoEdicao = false;
            idSelecionado = 0;
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            if (!ValidarCampos()) return;

            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query;

                    if (modoEdicao && idSelecionado > 0)
                    {
                        // Atualizar
                        query = @"UPDATE maquinas 
                                SET planta = @planta,
                                    nome_maquina = @nome,
                                    localizacao = @localizacao,
                                    status_operacional = @status
                                WHERE id_maquina = @id";
                    }
                    else
                    {
                        // Inserir
                        query = @"INSERT INTO maquinas 
                                (planta, nome_maquina, localizacao, status_operacional)
                                VALUES (@planta, @nome, @localizacao, @status)";
                    }

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@planta", cmbPlanta.SelectedItem.ToString());
                    cmd.Parameters.AddWithValue("@nome", txtNomeMaquina.Text.Trim());
                    cmd.Parameters.AddWithValue("@localizacao", txtLocalizacao.Text.Trim());
                    cmd.Parameters.AddWithValue("@status", cmbStatus.SelectedItem.ToString());

                    if (modoEdicao && idSelecionado > 0)
                        cmd.Parameters.AddWithValue("@id", idSelecionado);

                    cmd.ExecuteNonQuery();

                    MessageBox.Show("Máquina salva com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarMaquinas();
                    LimparCampos();
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnEditar_Click(object sender, EventArgs e)
        {
            if (dgvMaquinas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma máquina para editar!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            DataGridViewRow row = dgvMaquinas.SelectedRows[0];
            idSelecionado = Convert.ToInt32(row.Cells["ID"].Value);
            
            txtId.Text = idSelecionado.ToString();
            cmbPlanta.SelectedItem = row.Cells["Planta"].Value.ToString();
            txtNomeMaquina.Text = row.Cells["Nome Máquina"].Value.ToString();
            txtLocalizacao.Text = row.Cells["Localização"].Value?.ToString() ?? "";
            cmbStatus.SelectedItem = row.Cells["Status"].Value.ToString();

            HabilitarCampos(true);
            modoEdicao = true;
        }

        private void BtnExcluir_Click(object sender, EventArgs e)
        {
            if (dgvMaquinas.SelectedRows.Count == 0)
            {
                MessageBox.Show("Selecione uma máquina para excluir!", "Atenção",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            var result = MessageBox.Show(
                "Deseja realmente excluir esta máquina?\n\nAtenção: Isso pode afetar manutenções vinculadas!",
                "Confirmar Exclusão",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Warning
            );

            if (result == DialogResult.Yes)
            {
                try
                {
                    int id = Convert.ToInt32(dgvMaquinas.SelectedRows[0].Cells["ID"].Value);

                    using (MySqlConnection conn = ConexaoDB.ObterConexao())
                    {
                        string query = "DELETE FROM maquinas WHERE id_maquina = @id";
                        MySqlCommand cmd = new MySqlCommand(query, conn);
                        cmd.Parameters.AddWithValue("@id", id);
                        cmd.ExecuteNonQuery();
                    }

                    MessageBox.Show("Máquina excluída com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);

                    CarregarMaquinas();
                    LimparCampos();
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao excluir: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void BtnCancelar_Click(object sender, EventArgs e)
        {
            LimparCampos();
        }

        private void DgvMaquinas_SelectionChanged(object sender, EventArgs e)
        {
            if (dgvMaquinas.SelectedRows.Count > 0)
            {
                btnEditar.Enabled = true;
                btnExcluir.Enabled = true;
            }
            else
            {
                btnEditar.Enabled = false;
                btnExcluir.Enabled = false;
            }
        }

        private bool ValidarCampos()
        {
            if (cmbPlanta.SelectedIndex == -1)
            {
                MessageBox.Show("Selecione a planta!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                cmbPlanta.Focus();
                return false;
            }

            if (string.IsNullOrWhiteSpace(txtNomeMaquina.Text))
            {
                MessageBox.Show("Informe o nome da máquina!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                txtNomeMaquina.Focus();
                return false;
            }

            if (cmbStatus.SelectedIndex == -1)
            {
                MessageBox.Show("Selecione o status!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                cmbStatus.Focus();
                return false;
            }

            return true;
        }

        private void LimparCampos()
        {
            txtId.Clear();
            cmbPlanta.SelectedIndex = -1;
            txtNomeMaquina.Clear();
            txtLocalizacao.Clear();
            cmbStatus.SelectedIndex = 0;
            
            idSelecionado = 0;
            modoEdicao = false;
            
            HabilitarCampos(false);
            btnEditar.Enabled = false;
            btnExcluir.Enabled = false;
        }

        private void HabilitarCampos(bool habilitar)
        {
            cmbPlanta.Enabled = habilitar;
            txtNomeMaquina.Enabled = habilitar;
            txtLocalizacao.Enabled = habilitar;
            cmbStatus.Enabled = habilitar;
            btnSalvar.Enabled = habilitar;
        }
    }
}

using System;
using System.Drawing;
using System.IO;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmPerfil : Form
    {
        private PictureBox pbFotoPerfil;
        private TextBox txtNome, txtLogin, txtTelefone, txtEmail;
        private Label lblTipo;
        private Button btnSalvar, btnAlterarFoto, btnAlterarSenha, btnSair, btnGerenciarUsuarios;

        public FrmPerfil()
        {
            ConfigurarFormulario();
            CriarInterface();
            CarregarDadosUsuario();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Meu Perfil - Sistema de Manutenção";
            this.Size = new Size(700, 600);
            this.BackColor = Color.White;
            this.StartPosition = FormStartPosition.CenterScreen;
        }

        private void CriarInterface()
        {
            // Título
            Label lblTitulo = new Label
            {
                Text = "👤 Meu Perfil",
                Font = new Font("Segoe UI", 20, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                Location = new Point(30, 20),
                AutoSize = true
            };

            // Foto de Perfil
            pbFotoPerfil = new PictureBox
            {
                Location = new Point(280, 80),
                Size = new Size(140, 140),
                SizeMode = PictureBoxSizeMode.StretchImage,
                BorderStyle = BorderStyle.FixedSingle,
                BackColor = Color.FromArgb(230, 230, 230)
            };

            // Botão Alterar Foto
            btnAlterarFoto = new Button
            {
                Text = "📷 Alterar Foto",
                Location = new Point(265, 230),
                Size = new Size(170, 35),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAlterarFoto.FlatAppearance.BorderSize = 0;
            btnAlterarFoto.Click += BtnAlterarFoto_Click;

            // Campos
            int yPos = 290;

            // Nome
            Label lblNome = new Label
            {
                Text = "Nome:",
                Location = new Point(50, yPos),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtNome = new TextBox
            {
                Location = new Point(160, yPos),
                Size = new Size(450, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Login
            Label lblLogin = new Label
            {
                Text = "Login:",
                Location = new Point(50, yPos),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtLogin = new TextBox
            {
                Location = new Point(160, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10),
                Enabled = false,
                BackColor = Color.FromArgb(240, 240, 240)
            };
            yPos += 45;

            // Tipo
            Label lblTipoLabel = new Label
            {
                Text = "Tipo:",
                Location = new Point(50, yPos),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            lblTipo = new Label
            {
                Location = new Point(160, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 152, 219)
            };
            yPos += 45;

            // Telefone
            Label lblTelefone = new Label
            {
                Text = "Telefone:",
                Location = new Point(50, yPos),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtTelefone = new TextBox
            {
                Location = new Point(160, yPos),
                Size = new Size(200, 30),
                Font = new Font("Segoe UI", 10)
            };
            yPos += 45;

            // Email
            Label lblEmail = new Label
            {
                Text = "E-mail:",
                Location = new Point(50, yPos),
                Size = new Size(100, 25),
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };
            txtEmail = new TextBox
            {
                Location = new Point(160, yPos),
                Size = new Size(450, 30),
                Font = new Font("Segoe UI", 10)
            };

            // Botões
            btnSalvar = new Button
            {
                Text = "💾 Salvar",
                Location = new Point(50, 500),
                Size = new Size(140, 40),
                BackColor = Color.FromArgb(46, 204, 113),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnSalvar.FlatAppearance.BorderSize = 0;
            btnSalvar.Click += BtnSalvar_Click;

            btnAlterarSenha = new Button
            {
                Text = "🔒 Alterar Senha",
                Location = new Point(200, 500),
                Size = new Size(160, 40),
                BackColor = Color.FromArgb(230, 126, 34),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAlterarSenha.FlatAppearance.BorderSize = 0;
            btnAlterarSenha.Click += BtnAlterarSenha_Click;

            btnSair = new Button
            {
                Text = "🚪 Sair",
                Location = new Point(520, 500),
                Size = new Size(140, 40),
                BackColor = Color.FromArgb(231, 76, 60),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnSair.FlatAppearance.BorderSize = 0;
            btnSair.Click += BtnSair_Click;

            // Botão Gerenciar Usuários (só para Admin)
            btnGerenciarUsuarios = new Button
            {
                Text = "👥 Gerenciar Usuários",
                Location = new Point(370, 500),
                Size = new Size(140, 40),
                BackColor = Color.FromArgb(155, 89, 182),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 10, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand,
                Visible = SessaoUsuario.IsAdmin
            };
            btnGerenciarUsuarios.FlatAppearance.BorderSize = 0;
            btnGerenciarUsuarios.Click += BtnGerenciarUsuarios_Click;

            this.Controls.AddRange(new Control[] {
                lblTitulo, pbFotoPerfil, btnAlterarFoto,
                lblNome, txtNome,
                lblLogin, txtLogin,
                lblTipoLabel, lblTipo,
                lblTelefone, txtTelefone,
                lblEmail, txtEmail,
                btnSalvar, btnAlterarSenha, btnGerenciarUsuarios, btnSair
            });
        }

        private void CarregarDadosUsuario()
        {
            txtNome.Text = SessaoUsuario.NomeUsuario;
            txtLogin.Text = SessaoUsuario.LoginUsuario;
            lblTipo.Text = SessaoUsuario.TipoUsuario == "ADMIN" ? "ADMINISTRADOR" : "USUÁRIO";
            txtTelefone.Text = SessaoUsuario.Telefone ?? "";
            txtEmail.Text = SessaoUsuario.Email ?? "";

            if (SessaoUsuario.FotoPerfil != null && SessaoUsuario.FotoPerfil.Length > 0)
            {
                using (MemoryStream ms = new MemoryStream(SessaoUsuario.FotoPerfil))
                {
                    pbFotoPerfil.Image = Image.FromStream(ms);
                }
            }
        }

        private void BtnAlterarFoto_Click(object sender, EventArgs e)
        {
            OpenFileDialog dialog = new OpenFileDialog
            {
                Filter = "Imagens|*.jpg;*.jpeg;*.png;*.bmp",
                Title = "Selecione sua foto de perfil"
            };

            if (dialog.ShowDialog() == DialogResult.OK)
            {
                try
                {
                    pbFotoPerfil.Image = Image.FromFile(dialog.FileName);
                }
                catch (Exception ex)
                {
                    MessageBox.Show($"Erro ao carregar imagem: {ex.Message}", "Erro",
                        MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
        }

        private void BtnSalvar_Click(object sender, EventArgs e)
        {
            if (string.IsNullOrWhiteSpace(txtNome.Text))
            {
                MessageBox.Show("Informe seu nome!", "Validação",
                    MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            try
            {
                byte[] fotoBytes = null;
                if (pbFotoPerfil.Image != null)
                {
                    using (MemoryStream ms = new MemoryStream())
                    {
                        pbFotoPerfil.Image.Save(ms, System.Drawing.Imaging.ImageFormat.Jpeg);
                        fotoBytes = ms.ToArray();
                    }
                }

                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"UPDATE usuarios 
                                    SET nome = @nome, 
                                        telefone = @telefone, 
                                        email = @email, 
                                        foto_perfil = @foto
                                    WHERE id_usuario = @id";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    cmd.Parameters.AddWithValue("@nome", txtNome.Text.Trim());
                    cmd.Parameters.AddWithValue("@telefone", txtTelefone.Text.Trim());
                    cmd.Parameters.AddWithValue("@email", txtEmail.Text.Trim());
                    cmd.Parameters.AddWithValue("@foto", fotoBytes);
                    cmd.Parameters.AddWithValue("@id", SessaoUsuario.IdUsuario);

                    cmd.ExecuteNonQuery();

                    SessaoUsuario.AtualizarDados(txtNome.Text.Trim(), txtTelefone.Text.Trim(), 
                                                 txtEmail.Text.Trim(), fotoBytes);

                    MessageBox.Show("Perfil atualizado com sucesso!", "Sucesso",
                        MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao salvar: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnAlterarSenha_Click(object sender, EventArgs e)
        {
            FrmAlterarSenha frm = new FrmAlterarSenha();
            frm.ShowDialog();
        }

        private void BtnGerenciarUsuarios_Click(object sender, EventArgs e)
        {
            FrmGerenciarUsuarios frm = new FrmGerenciarUsuarios();
            frm.ShowDialog();
        }

        private void BtnSair_Click(object sender, EventArgs e)
        {
            DialogResult result = MessageBox.Show(
                "Deseja realmente sair do sistema?\n\nVocê será redirecionado para a tela de login.",
                "Confirmar Saída",
                MessageBoxButtons.YesNo,
                MessageBoxIcon.Question
            );

            if (result == DialogResult.Yes)
            {
                SessaoUsuario.Encerrar();
                this.DialogResult = DialogResult.OK;
                this.Close();
            }
        }
    }
}

using System;
using System.Drawing;
using System.Windows.Forms;
using MySql.Data.MySqlClient;

namespace SistemaManutencao
{
    public class FrmPrincipal : Form
    {
        // Painéis
        private Panel pnlSidebar;
        private Panel pnlContent;
        private Panel pnlProximaManutencao;

        // Botões do menu
        private Button btnManutencoes, btnHistorico, btnAjusteDatas;
        private Button btnConfiguracoes, btnConsulta, btnWhatsApp, btnPerfil, btnCadastroGeral;

        // Cards do menu principal
        private Panel cardManutencoes, cardHistorico, cardAjusteDatas;
        private Panel cardConfiguracoes, cardWhatsApp, cardCadastroGeral, cardVazio1;

        public FrmPrincipal()
        {
            ConfigurarFormulario();
            CriarSidebar();
            CriarConteudo();
            CarregarProximaManutencao();
        }

        private void ConfigurarFormulario()
        {
            this.Text = "Sistema de Manutenção - Menu Principal";
            this.Size = new Size(1600, 900);
            this.BackColor = Color.FromArgb(245, 245, 245);
            this.StartPosition = FormStartPosition.CenterScreen;
            this.WindowState = FormWindowState.Maximized;
        }

        private void CriarSidebar()
        {
            // Sidebar
            pnlSidebar = new Panel
            {
                Width = 250,
                Dock = DockStyle.Left,
                BackColor = Color.FromArgb(44, 62, 80)
            };

            // Header
            Label lblHeader = new Label
            {
                Text = "🔧\nSISTEMA DE\nMANUTENÇÃO",
                Location = new Point(0, 30),
                Size = new Size(250, 80),
                TextAlign = ContentAlignment.MiddleCenter,
                Font = new Font("Segoe UI", 14, FontStyle.Bold),
                ForeColor = Color.White
            };

            Panel separador1 = new Panel
            {
                Location = new Point(0, 120),
                Size = new Size(250, 2),
                BackColor = Color.FromArgb(52, 73, 94)
            };

            // Botões do Menu
            int yPos = 140;

            btnCadastroGeral = CriarBotaoMenu("📋 Cadastro Geral", yPos);
            btnCadastroGeral.Click += (s, e) => AbrirFormulario(new FrmCadastroGeral());
            yPos += 50;

            btnManutencoes = CriarBotaoMenu("🔧 Manutenções", yPos);
            btnManutencoes.Click += (s, e) => AbrirFormulario(new FrmCadastroManutencao());
            yPos += 50;

            btnHistorico = CriarBotaoMenu("📋 Histórico", yPos);
            btnHistorico.Click += (s, e) => AbrirFormulario(new FrmHistorico());
            yPos += 50;

            btnAjusteDatas = CriarBotaoMenu("📅 Ajuste de Datas", yPos);
            btnAjusteDatas.Click += (s, e) => AbrirFormulario(new FrmAjusteDatas());
            yPos += 50;

            btnConfiguracoes = CriarBotaoMenu("⚙️ Configurações", yPos);
            btnConfiguracoes.Click += (s, e) => AbrirFormulario(new FrmConfiguracoes());
            yPos += 50;

            btnConsulta = CriarBotaoMenu("🔍 Consulta Geral", yPos);
            btnConsulta.Click += BtnConsulta_Click;
            yPos += 50;

            // Separador
            Panel separador2 = new Panel
            {
                Location = new Point(10, yPos + 10),
                Size = new Size(230, 3),
                BackColor = Color.FromArgb(61, 86, 110)
            };
            yPos += 25;

            // Botão WhatsApp
            btnWhatsApp = new Button
            {
                Text = "📱 Enviar Resumo\nWhatsApp",
                Location = new Point(10, yPos),
                Size = new Size(230, 60),
                BackColor = Color.FromArgb(37, 211, 102),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                TextAlign = ContentAlignment.MiddleCenter,
                Cursor = Cursors.Hand
            };
            btnWhatsApp.FlatAppearance.BorderSize = 0;
            btnWhatsApp.Click += BtnWhatsApp_Click;
            btnWhatsApp.MouseEnter += BtnWhatsApp_MouseEnter;
            btnWhatsApp.MouseLeave += BtnWhatsApp_MouseLeave;

            // Botão Perfil (no rodapé) - Posição fixa
            btnPerfil = new Button
            {
                Text = "👤 Perfil",
                Location = new Point(10, 800),  // Posição fixa
                Size = new Size(230, 50),
                BackColor = Color.FromArgb(52, 73, 94),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand,
                Anchor = AnchorStyles.Bottom | AnchorStyles.Left | AnchorStyles.Right
            };
            btnPerfil.FlatAppearance.BorderSize = 0;
            btnPerfil.Click += BtnPerfil_Click;
            btnPerfil.MouseEnter += BtnPerfil_MouseEnter;
            btnPerfil.MouseLeave += BtnPerfil_MouseLeave;

            // Ajustar posição do botão quando o form redimensionar
            this.Resize += (s, e) =>
            {
                btnPerfil.Location = new Point(10, pnlSidebar.Height - 70);
            };

            pnlSidebar.Controls.AddRange(new Control[] {
                lblHeader, separador1,
                btnCadastroGeral, btnManutencoes, btnHistorico, btnAjusteDatas,
                btnConfiguracoes, btnConsulta, separador2, btnWhatsApp, btnPerfil
            });

            this.Controls.Add(pnlSidebar);
        }

        private Button CriarBotaoMenu(string texto, int y)
        {
            Button btn = new Button
            {
                Text = texto,
                Location = new Point(10, y),
                Size = new Size(230, 45),
                BackColor = Color.FromArgb(52, 73, 94),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Regular),
                FlatStyle = FlatStyle.Flat,
                TextAlign = ContentAlignment.MiddleLeft,
                Padding = new Padding(15, 0, 0, 0),
                Cursor = Cursors.Hand
            };

            btn.FlatAppearance.BorderSize = 0;
            btn.MouseEnter += (s, e) =>
            {
                btn.BackColor = Color.FromArgb(61, 86, 110);
                btn.Font = new Font("Segoe UI", 11, FontStyle.Bold);
            };
            btn.MouseLeave += (s, e) =>
            {
                btn.BackColor = Color.FromArgb(52, 73, 94);
                btn.Font = new Font("Segoe UI", 11, FontStyle.Regular);
            };

            return btn;
        }

        private void CriarConteudo()
        {
            pnlContent = new Panel
            {
                Location = new Point(250, 0),
                Size = new Size(this.ClientSize.Width - 250, this.ClientSize.Height),
                BackColor = Color.FromArgb(245, 245, 245),
                Anchor = AnchorStyles.Top | AnchorStyles.Bottom | AnchorStyles.Left | AnchorStyles.Right,
                AutoScroll = true
            };

            // Container para cards e quadrados lado a lado
            Panel containerPrincipal = new Panel
            {
                Location = new Point(30, 30),
                Size = new Size(pnlContent.Width - 60, pnlContent.Height - 60),
                BackColor = Color.Transparent,
                Anchor = AnchorStyles.Top | AnchorStyles.Bottom | AnchorStyles.Left | AnchorStyles.Right
            };

            // LADO ESQUERDO: CARDS (3x3)
            Panel containerCards = new Panel
            {
                Location = new Point(0, 0),
                Size = new Size(900, 800),
                BackColor = Color.Transparent
            };

            // Título
            Label lblTitulo = new Label
            {
                Text = "🔧 Acesso Rápido",
                Location = new Point(0, 0),
                Size = new Size(900, 40),
                Font = new Font("Segoe UI", 18, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64)
            };
            containerCards.Controls.Add(lblTitulo);

            // Grid de Cards 3x3
            int cardX = 0;
            int cardY = 60;
            int cardW = 280;
            int cardH = 200;
            int espacamento = 30;

            // Linha 1
            cardCadastroGeral = CriarCard("📋", "Cadastro Geral", "Cadastre máquinas, peças e funcionários", Color.FromArgb(26, 188, 156), cardX, cardY, 
                () => AbrirFormulario(new FrmCadastroGeral()));
            
            Panel cardConsulta = CriarCard("🔍", "Consulta Geral", "Consulte tudo que foi cadastrado", Color.FromArgb(52, 152, 219), cardX + cardW + espacamento, cardY, 
                () => BtnConsulta_Click(null, null));
            
            cardManutencoes = CriarCard("🔧", "Manutenções", "Registre e consulte manutenções", Color.FromArgb(155, 89, 182), cardX + (cardW + espacamento) * 2, cardY, 
                () => AbrirFormulario(new FrmCadastroManutencao()));

            // Linha 2
            cardY += cardH + espacamento;
            cardHistorico = CriarCard("📋", "Histórico", "Visualize o histórico completo", Color.FromArgb(52, 73, 94), cardX, cardY, 
                () => AbrirFormulario(new FrmHistorico()));
            cardAjusteDatas = CriarCard("📅", "Ajuste de Datas", "Ajuste datas de manutenções", Color.FromArgb(230, 126, 34), cardX + cardW + espacamento, cardY, 
                () => AbrirFormulario(new FrmAjusteDatas()));
            cardConfiguracoes = CriarCard("⚙️", "Configurações", "Configure o sistema", Color.FromArgb(149, 165, 166), cardX + (cardW + espacamento) * 2, cardY, 
                () => AbrirFormulario(new FrmConfiguracoes()));

            // Linha 3
            cardY += cardH + espacamento;
            cardWhatsApp = CriarCard("📱", "WhatsApp", "Envie resumo via WhatsApp", Color.FromArgb(37, 211, 102), cardX, cardY, 
                () => BtnWhatsApp_Click(null, null));
            cardVazio1 = CriarCard("📊", "Relatórios", "Em breve: relatórios completos", Color.FromArgb(189, 195, 199), cardX + cardW + espacamento, cardY, 
                () => MessageBox.Show("Funcionalidade em desenvolvimento", "Em breve", MessageBoxButtons.OK, MessageBoxIcon.Information));

            containerCards.Controls.AddRange(new Control[] {
                cardCadastroGeral, cardConsulta, cardManutencoes,
                cardHistorico, cardAjusteDatas, cardConfiguracoes,
                cardWhatsApp, cardVazio1
            });

            // LADO DIREITO: QUADRADOS DE ESTATÍSTICAS (ANCORADO NA DIREITA)
            Panel containerQuadrados = new Panel
            {
                Width = 370,
                Height = 800,
                BackColor = Color.Transparent,
                Anchor = AnchorStyles.Top | AnchorStyles.Right // Ancorado na direita
            };
            
            // Posicionar na borda direita
            containerQuadrados.Left = pnlContent.Width - containerQuadrados.Width - 30;

            Label lblEstatisticas = new Label
            {
                Text = "📊 Estatísticas",
                Location = new Point(0, 0),
                Size = new Size(250, 40),
                Font = new Font("Segoe UI", 18, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64)
            };
            containerQuadrados.Controls.Add(lblEstatisticas);

            // Estatísticas - criar ANTES do botão para poder referenciar
            Panel quadrado1 = CriarQuadradoEstatistica("ATRASADO", "0", "Manutenções", Color.FromArgb(231, 76, 60), 0, 60);
            Panel quadrado2 = CriarQuadradoEstatistica("HOJE", "0", "Manutenções", Color.FromArgb(255, 193, 7), 0, 180);
            Panel quadrado3 = CriarQuadradoEstatistica("PLANEJADO", "0", "Manutenções", Color.FromArgb(52, 152, 219), 0, 300);
            Panel quadrado4 = CriarQuadradoEstatistica("REALIZADAS", "0", "Total de manutenções", Color.FromArgb(46, 204, 113), 0, 420);

            // BOTÃO DE ATUALIZAR (acima dos cards)
            Button btnAtualizar = new Button
            {
                Text = "🔄 Atualizar",
                Location = new Point(210, 5), // Ajustado para nova largura
                Size = new Size(150, 35),
                BackColor = Color.FromArgb(52, 152, 219),
                ForeColor = Color.White,
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                FlatStyle = FlatStyle.Flat,
                Cursor = Cursors.Hand
            };
            btnAtualizar.FlatAppearance.BorderSize = 0;
            btnAtualizar.Click += (s, e) => {
                // Recarregar estatísticas
                CarregarEstatisticas(quadrado1, quadrado2, quadrado3, quadrado4);
                CarregarProximaManutencao();
                MessageBox.Show("✅ Estatísticas atualizadas!", "Atualizado", MessageBoxButtons.OK, MessageBoxIcon.Information);
            };
            containerQuadrados.Controls.Add(btnAtualizar);

            // Próxima Manutenção
            pnlProximaManutencao = new Panel
            {
                Location = new Point(0, 540),
                Size = new Size(360, 180), // Mantido em 360 devido ao novo espaço
                BackColor = Color.White,
                BorderStyle = BorderStyle.None
            };

            Label lblProximaTitulo = new Label
            {
                Text = "📅 PLANEJADO - 21 DIAS",
                Location = new Point(10, 10),
                Size = new Size(340, 30),
                Font = new Font("Segoe UI", 12, FontStyle.Bold),
                ForeColor = Color.White,
                BackColor = Color.FromArgb(52, 152, 219),
                TextAlign = ContentAlignment.MiddleCenter
            };

            Label lblProximaInfo = new Label
            {
                Name = "lblProximaInfo",
                Text = "Carregando...",
                Location = new Point(10, 50),
                Size = new Size(340, 120),
                Font = new Font("Segoe UI", 10),
                ForeColor = Color.FromArgb(52, 73, 94)
            };

            pnlProximaManutencao.Controls.AddRange(new Control[] { lblProximaTitulo, lblProximaInfo });

            containerQuadrados.Controls.AddRange(new Control[] { quadrado1, quadrado2, quadrado3, quadrado4, pnlProximaManutencao });

            // Carregar estatísticas reais
            CarregarEstatisticas(quadrado1, quadrado2, quadrado3, quadrado4);

            containerPrincipal.Controls.AddRange(new Control[] { containerCards, containerQuadrados });
            pnlContent.Controls.Add(containerPrincipal);
            this.Controls.Add(pnlContent);
            
            // Reposicionar cards quando a janela redimensionar
            this.Resize += (s, e) => {
                if (containerQuadrados != null && containerPrincipal != null)
                {
                    containerQuadrados.Left = containerPrincipal.Width - containerQuadrados.Width - 30;
                }
            };
            
            // Reposicionar após carregar
            this.Load += (s, e) => {
                if (containerQuadrados != null && containerPrincipal != null)
                {
                    containerQuadrados.Left = containerPrincipal.Width - containerQuadrados.Width - 30;
                }
            };
        }

        private Panel CriarQuadradoEstatistica(string titulo, string valor, string legenda, Color cor, int x, int y)
        {
            Panel quadrado = new Panel
            {
                Location = new Point(x, y),
                Size = new Size(360, 100),
                BackColor = Color.White,
                BorderStyle = BorderStyle.None
            };

            // Borda customizada
            quadrado.Paint += (s, e) =>
            {
                using (Pen p = new Pen(Color.FromArgb(220, 220, 220), 2))
                {
                    e.Graphics.DrawRectangle(p, 0, 0, quadrado.Width - 1, quadrado.Height - 1);
                }
            };

            // Faixa colorida
            Panel faixa = new Panel
            {
                Location = new Point(0, 0),
                Size = new Size(8, 100),
                BackColor = cor
            };

            Label lblTitulo = new Label
            {
                Text = titulo,
                Location = new Point(20, 15),
                Size = new Size(200, 25),
                Font = new Font("Segoe UI", 11, FontStyle.Bold),
                ForeColor = Color.FromArgb(127, 140, 141)
            };

            Label lblValor = new Label
            {
                Text = valor,
                Location = new Point(240, 10),
                Size = new Size(100, 50),
                Font = new Font("Segoe UI", 32, FontStyle.Bold),
                ForeColor = cor,
                TextAlign = ContentAlignment.MiddleRight
            };

            Label lblLegenda = new Label
            {
                Text = legenda,
                Location = new Point(20, 65),
                Size = new Size(320, 20),
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.FromArgb(149, 165, 166)
            };

            quadrado.Controls.AddRange(new Control[] { faixa, lblTitulo, lblValor, lblLegenda });

            return quadrado;
        }

        private void CarregarEstatisticas(Panel quadrado1, Panel quadrado2, Panel quadrado3, Panel quadrado4)
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    // Atrasadas
                    string queryAtrasadas = "SELECT COUNT(*) FROM manutencoes WHERE situacao = 'ATRASADO'";
                    MySqlCommand cmdAtrasadas = new MySqlCommand(queryAtrasadas, conn);
                    int atrasadas = Convert.ToInt32(cmdAtrasadas.ExecuteScalar());
                    ((Label)quadrado1.Controls[2]).Text = atrasadas.ToString();

                    // HOJE - Pega manutenções planejadas para hoje que ainda não foram realizadas
                    string queryHoje = @"SELECT COUNT(*) FROM manutencoes 
                                        WHERE DATE(data_planejada) = CURDATE() 
                                        AND situacao != 'REALIZADO'";
                    MySqlCommand cmdHoje = new MySqlCommand(queryHoje, conn);
                    int hoje = Convert.ToInt32(cmdHoje.ExecuteScalar());
                    ((Label)quadrado2.Controls[2]).Text = hoje.ToString();

                    // Planejadas
                    string queryPlanejadas = "SELECT COUNT(*) FROM manutencoes WHERE situacao = 'PLANEJADO'";
                    MySqlCommand cmdPlanejadas = new MySqlCommand(queryPlanejadas, conn);
                    int planejadas = Convert.ToInt32(cmdPlanejadas.ExecuteScalar());
                    ((Label)quadrado3.Controls[2]).Text = planejadas.ToString();

                    // Realizadas este mês
                    string queryRealizadas = @"SELECT COUNT(*) FROM manutencoes 
                                              WHERE situacao = 'REALIZADO'";
                    MySqlCommand cmdRealizadas = new MySqlCommand(queryRealizadas, conn);
                    int realizadas = Convert.ToInt32(cmdRealizadas.ExecuteScalar());
                    ((Label)quadrado4.Controls[2]).Text = realizadas.ToString();
                    
                    // Atualizar legenda para "Total"
                    ((Label)quadrado4.Controls[3]).Text = "Total de manutenções";
                }
            }
            catch
            {
                // Silencioso
            }
        }

        private Panel CriarCard(string icone, string titulo, string descricao, Color corBorda, int x, int y, Action acao)
        {
            Panel card = new Panel
            {
                Location = new Point(x, y),
                Size = new Size(280, 200),
                BackColor = Color.White,
                Cursor = Cursors.Hand,
                BorderStyle = BorderStyle.None
            };

            // Faixa superior colorida com cantos arredondados
            Panel faixa = new Panel
            {
                Location = new Point(0, 0),
                Size = new Size(280, 8),
                BackColor = Color.Transparent,  // Transparente para o Paint funcionar
                Cursor = Cursors.Hand
            };

            // Desenhar faixa com cantos arredondados
            faixa.Paint += (s, e) =>
            {
                e.Graphics.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
                
                // Criar path com cantos arredondados apenas em cima
                using (System.Drawing.Drawing2D.GraphicsPath path = new System.Drawing.Drawing2D.GraphicsPath())
                {
                    int raio = 12;
                    int largura = faixa.Width;
                    int altura = faixa.Height;
                    
                    // Começar no canto superior esquerdo (após o arco)
                    path.AddArc(0, 0, raio * 2, raio * 2, 180, 90);
                    
                    // Linha superior
                    path.AddLine(raio, 0, largura - raio, 0);
                    
                    // Canto superior direito arredondado
                    path.AddArc(largura - raio * 2, 0, raio * 2, raio * 2, 270, 90);
                    
                    // Linha direita (reta)
                    path.AddLine(largura, raio, largura, altura);
                    
                    // Linha inferior (reta)
                    path.AddLine(largura, altura, 0, altura);
                    
                    // Linha esquerda (reta)
                    path.AddLine(0, altura, 0, raio);
                    
                    path.CloseFigure();

                    // Preencher com a cor
                    using (SolidBrush brush = new SolidBrush(corBorda))
                    {
                        e.Graphics.FillPath(brush, path);
                    }
                }
            };

            Label lblIcone = new Label
            {
                Text = icone,
                Location = new Point(100, 35),
                Size = new Size(80, 60),
                Font = new Font("Segoe UI", 40),
                ForeColor = corBorda,
                TextAlign = ContentAlignment.MiddleCenter,
                Cursor = Cursors.Hand
            };

            Label lblTitulo = new Label
            {
                Text = titulo,
                Location = new Point(20, 105),
                Size = new Size(240, 30),
                Font = new Font("Segoe UI", 13, FontStyle.Bold),
                ForeColor = Color.FromArgb(52, 58, 64),
                TextAlign = ContentAlignment.MiddleCenter,
                Cursor = Cursors.Hand
            };

            Label lblDesc = new Label
            {
                Text = descricao,
                Location = new Point(20, 140),
                Size = new Size(240, 40),
                Font = new Font("Segoe UI", 9),
                ForeColor = Color.FromArgb(127, 140, 141),
                TextAlign = ContentAlignment.TopCenter,
                Cursor = Cursors.Hand
            };

            // BORDA COM HOVER
            card.Paint += (s, e) =>
            {
                e.Graphics.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;

                Color borderColor = (card.Tag?.ToString() == "hover") ? corBorda : Color.FromArgb(200, 200, 200);
                int borderWidth = (card.Tag?.ToString() == "hover") ? 2 : 1;

                using (Pen pen = new Pen(borderColor, borderWidth))
                {
                    System.Drawing.Drawing2D.GraphicsPath borderPath = new System.Drawing.Drawing2D.GraphicsPath();
                    int r = 12;
                    borderPath.AddArc(1, 1, r * 2, r * 2, 180, 90);
                    borderPath.AddArc(card.Width - r * 2 - 2, 1, r * 2, r * 2, 270, 90);
                    borderPath.AddArc(card.Width - r * 2 - 2, card.Height - r * 2 - 2, r * 2, r * 2, 0, 90);
                    borderPath.AddArc(1, card.Height - r * 2 - 2, r * 2, r * 2, 90, 90);
                    borderPath.CloseFigure();
                    e.Graphics.DrawPath(pen, borderPath);
                }
            };

            // HOVER - LIGAR
            EventHandler mouseEnter = (s, e) =>
            {
                card.Tag = "hover";
                card.Invalidate();
            };

            // HOVER - DESLIGAR
            EventHandler mouseLeave = (s, e) =>
            {
                card.Tag = null;
                card.Invalidate();
            };

            // Aplicar eventos em TODOS os controles
            card.MouseEnter += mouseEnter;
            card.MouseLeave += mouseLeave;
            faixa.MouseEnter += mouseEnter;
            faixa.MouseLeave += mouseLeave;
            lblIcone.MouseEnter += mouseEnter;
            lblIcone.MouseLeave += mouseLeave;
            lblTitulo.MouseEnter += mouseEnter;
            lblTitulo.MouseLeave += mouseLeave;
            lblDesc.MouseEnter += mouseEnter;
            lblDesc.MouseLeave += mouseLeave;

            // CLICK
            card.Click += (s, e) => acao();
            faixa.Click += (s, e) => acao();
            lblIcone.Click += (s, e) => acao();
            lblTitulo.Click += (s, e) => acao();
            lblDesc.Click += (s, e) => acao();

            card.Controls.Add(faixa);
            card.Controls.Add(lblIcone);
            card.Controls.Add(lblTitulo);
            card.Controls.Add(lblDesc);

            return card;
        }

        private void CarregarProximaManutencao()
        {
            try
            {
                using (MySqlConnection conn = ConexaoDB.ObterConexao())
                {
                    string query = @"
                        SELECT 
                            m.data_planejada,
                            maq.nome_maquina,
                            p.nome_peca,
                            maq.planta,
                            m.situacao,
                            DATEDIFF(m.data_planejada, CURDATE()) AS dias
                        FROM manutencoes m
                        INNER JOIN maquinas maq ON m.id_maquina = maq.id_maquina
                        INNER JOIN pecas p ON m.id_peca = p.id_peca
                        WHERE m.situacao IN ('PLANEJADO', 'HOJE', 'ATRASADO')
                        ORDER BY m.data_planejada ASC
                        LIMIT 1";

                    MySqlCommand cmd = new MySqlCommand(query, conn);
                    MySqlDataReader reader = cmd.ExecuteReader();

                    if (reader.Read())
                    {
                        string situacao = reader["situacao"].ToString();
                        Color corHeader = Color.FromArgb(255, 193, 7); // HOJE
                        Color corFundo = Color.FromArgb(255, 249, 196);
                        string textoHeader = "⏰ MANUTENÇÃO HOJE";

                        if (situacao == "PLANEJADO")
                        {
                            corHeader = Color.FromArgb(52, 152, 219);
                            corFundo = Color.FromArgb(179, 229, 252);
                            int dias = Convert.ToInt32(reader["dias"]);
                            textoHeader = $"📅 PLANEJADO - {dias} DIAS";
                        }
                        else if (situacao == "ATRASADO")
                        {
                            corHeader = Color.FromArgb(231, 76, 60);
                            corFundo = Color.FromArgb(255, 205, 210);
                            int dias = Math.Abs(Convert.ToInt32(reader["dias"]));
                            textoHeader = $"❌ ATRASADO - {dias} DIAS";
                        }

                        pnlProximaManutencao.BackColor = corFundo;
                        pnlProximaManutencao.Paint += (s, e) =>
                        {
                            using (Pen p = new Pen(corHeader, 3))
                            {
                                e.Graphics.DrawRectangle(p, 0, 0, pnlProximaManutencao.Width - 1, pnlProximaManutencao.Height - 1);
                            }
                        };

                        Panel header = new Panel
                        {
                            Location = new Point(0, 0),
                            Size = new Size(360, 40),
                            BackColor = corHeader
                        };

                        Label lblHeader = new Label
                        {
                            Text = textoHeader,
                            Dock = DockStyle.Fill,
                            TextAlign = ContentAlignment.MiddleCenter,
                            Font = new Font("Segoe UI", 11, FontStyle.Bold),
                            ForeColor = situacao == "HOJE" ? Color.Black : Color.White
                        };

                        header.Controls.Add(lblHeader);

                        Label lblConteudo = new Label
                        {
                            Text = $"Data: {Convert.ToDateTime(reader["data_planejada"]):dd/MM/yyyy}\n" +
                                   $"Máquina: {reader["nome_maquina"]}\n" +
                                   $"Peça: {reader["nome_peca"]}\n" +
                                   $"Planta: {reader["planta"]}",
                            Location = new Point(15, 50),
                            Size = new Size(330, 120),
                            Font = new Font("Segoe UI", 10),
                            ForeColor = Color.FromArgb(44, 62, 80)
                        };

                        pnlProximaManutencao.Controls.Clear();
                        pnlProximaManutencao.Controls.AddRange(new Control[] { header, lblConteudo });
                    }
                    reader.Close();
                }
            }
            catch
            {
                // Silencioso
            }
        }

        private void AbrirFormulario(Form formulario)
        {
            formulario.ShowDialog();
        }

        private void BtnWhatsApp_MouseEnter(object sender, EventArgs e)
        {
            btnWhatsApp.BackColor = Color.FromArgb(31, 178, 85);
        }

        private void BtnWhatsApp_MouseLeave(object sender, EventArgs e)
        {
            btnWhatsApp.BackColor = Color.FromArgb(37, 211, 102);
        }

        private void BtnPerfil_MouseEnter(object sender, EventArgs e)
        {
            btnPerfil.BackColor = Color.FromArgb(61, 86, 110);
        }

        private void BtnPerfil_MouseLeave(object sender, EventArgs e)
        {
            btnPerfil.BackColor = Color.FromArgb(52, 73, 94);
        }

        private void BtnPerfil_Click(object sender, EventArgs e)
        {
            if (!SessaoUsuario.EstaLogado)
            {
                MessageBox.Show(
                    "Você está no modo sem login.\n\n" +
                    "Para acessar o perfil, faça login no sistema.",
                    "Login Necessário",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Information
                );
                return;
            }

            FrmPerfil frm = new FrmPerfil();
            if (frm.ShowDialog() == DialogResult.OK)
            {
                // Usuário clicou em Sair
                Application.Restart();
            }
        }

        private void BtnWhatsApp_Click(object sender, EventArgs e)
        {
            try
            {
                MessageBox.Show("Enviando resumo via WhatsApp...", "WhatsApp", 
                    MessageBoxButtons.OK, MessageBoxIcon.Information);
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Erro ao enviar WhatsApp: {ex.Message}", "Erro",
                    MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void BtnConsulta_Click(object sender, EventArgs e)
        {
            try
            {
                FrmConsultaGeral frm = new FrmConsultaGeral();
                frm.ShowDialog();
            }
            catch (Exception ex)
            {
                MessageBox.Show(
                    "❌ Erro ao abrir Consulta Geral\n\n" +
                    $"Detalhes: {ex.Message}\n\n" +
                    "Certifique-se de que o arquivo FrmConsultaGeral.cs\n" +
                    "foi adicionado ao projeto e compilado corretamente.",
                    "Erro",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error
                );
            }
        }
    }
}



