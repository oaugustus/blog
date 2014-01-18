Escrevendo um Plugin para o GLPI
================================

A partir de uma necessidade prática, vamos explorar a estrutura de Plugins GLPI 
e mostrar como estender a funcionalidade desse software de gerenciamento de ativos. 

**Palavras-chave**: GLPI, PHP, Plugin

## 1 Objetivos

### 1.1 Apresentação do GLPI

GLPI significa Gerenciador Livre de Parque de TI. É uma aplicação livre escrita em PHP + MySQL, desenvolvida por uma equipe de programadores franceses. Através de sua interface WEB, GLPI permite gerenciar recursos de TI, licenças de software, insumos, fornecedores, reserva de equipamentos. GLPI também gerencia "Helpdesk" através da criação e gestão de tiquets. 

Muitas vezes, o GLPI é acoplado a outro aplicativo chamado OCS Inventory, que também é desenvolvido por franceses. Com um agente implantado em cada estação de trabalho do usuário, a dupla GLPI-OCS também permite a implementação de aplicações remotas. Esta combinação faz dessas ferramentas uma poderosa solução. 

Neste artigo, vamos nos concentrar apenas no GLPI, em particular, mostrar como enriquecê-lo com informações personalizadas.

### 1.2 Nossas necessidades adicionais

#### 1.2.1 Adicionando campos "abertos"

Quando os técnicos do "helpdesk", são solicitados por usuários eles criam o que chamamos de "Tiquets" na interface do GLPI. Informações que podem ou devem ser preenchidas são bem completas (Status, prioridade, autor, etc...) assim como exibido na tela abaixo:

[img]

No entanto, os campos fornecidos nem sempre são suficientes em termos de necessidade de "trabalho"

Por exemplo, o formulário não captura informações "abertas", como o número de série do PDA, ou um número de de tiquet "externo"  quando o problema é enviado para um fornecedor terceiro que nos informa seu próprio número de tiquete. Poderíamos dar uma infinidade de exemplos, e também tentar generalizar a nossa primeira necessidade que consiste em adicionar campos adicionais na criação de um bilhete.

#### 1.2.2 Separar o problema da solução

Ao criar o bilhete o autor descreve o problema no espaço fornecido para isso, depois cada participante que tenta resolver o problema acrescenta um novo "andamento" no tiquete. Até que alguém resolva o problema. Neste caso, gostaríamos que a solução fosse claramente identificada, e subsquentemente ela pode ser exibida.

#### 1.2.3 Screenshots associados aos tiquetes

Existe a tela do formulário para arquivos associados a um tiquete (veja a imagem acima). No entanto, se esses arquivos são imagens (por exemplo, screenshots contendo as mensagens de erro do chamado), nós gostaríamos de ver essas imagens ao ver o bilhete.

## 2. Solução proposta: escrever um plugin

Tomamos como base as necessidades descritas anteriormente para te mostrar como escreve um plugin (extensão) para o GLPI. De fator, o GLPI oferece a possibilidade, através de um mecanismo de "hooks" (ganchos), de enriquecer o seu tratamento pelo código adicional.

Claro, existem algumas regras a se seguir e algumas limitações. INformações sobre a estrutura de plugins estão disponívels no endereço: [LINK_DOC]. Um estudo dos fontes também te permite complementar essa informação. 

    Nota: Neste artigo, vamos assumir que o código-fonte do GLPI foi instalado no diretório / opt / glpi e vamos chamar esse diretório $ ROOT.

### 2.1 Os "ganchos" do GLPI

Scripts do GLPI usam um certas variáveis globais, incluindo uma matriz associativa multidimensional chamada $PLUGIN_HOOKS. Em diferentes estágios dos objetos de gestão (tiquetes, forncedores, equipamentos...) GLPI invoca a lista de todos os plugins que registram uma função para essa etapa. 

Os ganchos podem ser classificados em várias categorias. A tabela a seguir lista os "ganchos" que afetam a interface web.

+ **menu_entry** (booleano): exibe uma opção no menu central > Plugins;
+ **submenu_entry** (texto): página exibida quando se seleciona o nome do plugin no menu Central > Plugins;
+ **helpdesk_menu_entry** (booleano): exibe o nome do plugin na guia de menu de acompanhamento "plugins" ao visualizar o tiquete;
+ **headings** (função): retorna o nome da função a ser chamada quando a guia de menu é selecionada (ver § 4.3.1);
+ **headings_action** (função): retorna o nome da função a ser chamado quando a guia do menu é selecionado (ver § 4.3.1);
+ **config_page** (texto): o formulário de configuração do plugin (ver 4.2)
+ **add_javascript** (nome do arquivo): nome do arquivo javascript para ser incluído nas páginas (o nome é relativo ao diretório do plugin);
+ **add_css** (nome do arquivo): nome do arquivo CSS a ser incluído nas páginas (o nome é em relação ao diretório do plugin);
+ **central_action** (função): retorna o código HTML para exibir a seleção de guia Pugins "central" (ver a tela # 2 abaixo);
+ **display_planning** (função): define a exibição na programação;
+ **planning_populate** (função): preenche uma matriz de objetos no planejamento;
+ **reports** (função): fornece a possibilidade de complementar as páginas do plugin de relatórios (menu Central > Ferramentas > Relatórios);
+ **user_preferences** (função): retorna o código HTML que permite ao usuário alterar as preferências para este plugin;
+ **stats** (função): possibilita adicionar o plugin na tabela de estatísticas (menu central > Suporte > Estatísticas);

A tela abaixo mostra a exibição produzida pelo gancho central_action:

[IMG]

A tabela a seguir lista os "ganchos" que são chamados ao manipular objetos internos do GLPI:

+ **pre_item_add, item_add** (funções): funções que são invocadas antes e após da adição de um novo objeto;
+ **pre_item_delete, item_delete** (funções): funções que são invocadas antes e a exclusão de um objeto;
+ **pre_item_update, item_update** (funções): funções que são invocadas antes da atualização de um objeto;
+ **pre_item_purge, item_purge** (funções): funções que são invocadas antes e após a movimentação de um objeto para a lixeira;
+ **pre_item_restore, item_restore** (funções): funções que são invocadas antes e após a restauração de um objeto da lixeira;
+ **use_massive_action** (booleano): indica se o plugin deve gerenciar ações de modificar, apagar, remover ou restaurar invocado a partir de formas que exibem listas de objetos;
+ **item_transfer** (função): função executada após a conclusão da transferência de um item para outro;
+ **rule_matched** (função): funções chamadas durante o processamento de cada regra;

Finalmente, a tabela a seguir lista os "ganchos" que afetam vários serviços:

+ **cron** (frequencia): dá a periodicidade da chamada de função que define o plugin tarefa agendada codificado na função cujo o nome é cron_plugin_nom_plugin ();
+ **redirect_page** (texto): nome do script para invocar redirecionamentos;
+ **init_session** (função): função chamada ao criar uma sessão (quando se autentica na interface web do GLPI);
+ **change_profile** (função): função chamada quando o usuário muda a sua identidade na interface;
+ **change_entity** (função): função chamada quando o usuário altera o uma entidade GLPI;
+ **restrict_ldap_auth, retrieve_more_data_from_ldap** (função): funções apresentadas quando se olha para o diretório de um usuário;

### 2.2. Outras variáveis globais acessadas por plug-ins

Além das variáveis $ PLUGIN_HOOKS globais, GLPI define um número de outras variáveis globais utilizadas por plug-ins. Lista de variáveis:

+ **$DB**: instância da classe DB para acessar o banco de dados;
+ **$CFG_GLPI**: parâmetros globais GLPI modificável pelo menu "Configurações" da interface Web;
+ **$LANG**: tabela de traduções;
+ **$NEEDED_ITEMS: tabela de classes de objetos utilizadas por um script;
+ **GLPI_ROOT**: (constante Global) correspondente ao diretório raiz da interface Web;

### 2.3. Mensagens e localização

Para cada idioma suportado pelo nosso plugin, deve existir um arquivo localizado no diretório local do plugin. O nome do arquivo deve ser no padrão I18n.

Esses arquivos contém linhas que alimentam o array $LANG['nom_plugin'], por exemplo:

    $LANG['nome_do_plugin']['palavra-chave'][indice] = "Valor"

Valores, 'palavras-chave' e 'índice' são abritrariamente definidos pelo plugin.

### 2.4 Suporte à API fornecida pelo GLPI

#### 2.4.1 Classes usuais - de acesso às tabelas do banco de dados

Entre as classes definidas nos arquivos $ROOT/inc/*.class.php, a mais utilizada é provavelmente o CommonDBTM, definida em $ROOT/inc/commondbtm.class.php. Essa classe implementa as funções de conexão, acesso, manipulação de banco de dados. 

Para cada novo objeto que é armazenado, será necessário acessar essa classe para acessar os métodos add(), update(), delete(), etc... 

Observe também a classe glpi_phpmailer situada em $ROOT/inc/mailing.class.php que permite enviar emails.

#### 2.4.2 Funções globais

O conteúdo do diretório $ROOT/inc também possui arquivos cujo nome tem a forma *.function.php. Esses arquivos contém funções globais reutilizáveis no código do nosso plugin. O arquivo display.function.php por exemplo, contém os métodos commonHeader() e commonFooter() que exibem respectivamente o cabeçalho e o rodapé da interface web.

### 3. Arquitetura do nosso plug-in 

É isso aí, agora que já vimos bastante a parte teórica, está na hora de arregaçar as mangas e implementar nosso plugin! 

Antes de qualquer coisa, precisamos batizar o nosso plugin. Decidimos que ele vai ser chamado de "miniatura", ainda que ele vá fazer mais do que exibir miniaturas na tela de tiquetes.

#### 3.1. Árvore de arquivos e diretórios 

De acordo com a documentação do [GLPI1] e [GLPI4], devemos criar o diretório $ROOT/plugins/miniatura e dentro desse diretório teremos a seguinte estrutura de diretórios e arquivos:

+ ./inc: conjunto de scripts incluídos por outros scripts de plugins;
+ ./front: scripts invocados diretamente através da interface Web;
+ ./pic: imagens próprias do plugin;
+ ./doc: contém os arquivos Changelog.txt, Readme.txt e Roadmap.txt;
+ ./locales: contém arquivos de traduções (ver § 2.3);
+ ./index.php: um arquivo vazio ou um arquivo que contém o código HTML é exibido ao se clicar no item do plug-in no menu "Plugins" no menu principal;
+ ./setup.php: (obrigatório) define as funções obrigatórias;
+ ./hook.php: define os hooks do plugin;

#### 3.2 Criação do esqueleto do nosso plugin

Temos todas as informações para começar:

    $ cd /opt/glpi/plugins 
    $ mkdir miniatura

#### 3.2.1. Banco de dados de armazenamento 

Como afirmado em [GLPI1], a informação complementar sobre os tiquetes e é gerenciada pelo nosso plugin deve ser armazenada em tabelas cujo nome é prefixado por glpi_plugin_miniatura_. 
Vamos criar a tabela de glpi_plugin_miniatura_extinfo (como "informação estendida") que irá conter todas as informações complementares associadas a um tiquete. Para cada tiquete, vamos adicionar os seguintes campos:


 + **miniatura_extticket**: contém o valor da passagem externa fornecida opcionalmente por um fornecedor externo;
 + **miniatura_solution**: contém a solução que ajudou a fechar o tiquete;
 + **id_tracking**: identificador (ID), que estão ligadas as informações sobre tiquetes. Esta é uma referência ao campo ID na tabela glpi_tracking;
 + **ID**: a classe CommonDBTM exige que cada tabela tenha um campo inteiro, que é a chave primária auto-incremento;
 
Como dissemos acima, os autores do GLPI tiveram a boa idéia para fornecer uma classe base, chamada CommonDBTM que encapsula o acesso ao SGBD. A administração de nossa nova tabela glpi_plugin_miniatura_extinfo é garantida pela classe plugin_Miniatura definida no arquivo $ROOT/plugins/miniatura/inc/plugin_miniatura_classes.php:

    <?php
        if (!defined('GLPI_ROOT')) {
            die("Sorry. You can't access directly to this file");
        }
        class plugin_Miniatura extends commonDBTM {
            public function plugin_Miniatura() {
                $this->table = "glpi_plugin_miniatura_extinfo"; 
                $this->type = PLUGIN_MINIATURA_TYPE;
            }
        }
    ?>
 
Observação: não se apresse para criar a tabela de banco de dados glpi_plugin_miniatira_extinfo. Com um cliente SQL você provavelmente faria um "CREATE TABLE". Veremos no item §3.3.1, um mecanismo de instalação e desinstalação proposto pelo GLPI que é responsável por este papel. 
Então vamos criar um arquivo setup.php que deve conter pelo menos seis funções obrigatórias.

#### 3.2.2. Função plugin_version_miniatura()

A primeira função simplesmente define o nome e a versão do plugin. O nome do autor e o endereço do seu site são opcionais:

    function plugin_version_miniatura()
    {
        return array(
            'name' => "Vignette", 'minGlpiVersion' => '0.71', 
            'version' => '1.0',
            'author' => 'jd',
            'homepage' => 'http://www.maje.biz');    
    }
 
Indicamos o número mínimo da versão GLPI mínimo esperada. No Menu Central > Configurações > Plugins exibe a lista de plugins disponíveis e as informações retornadas por esta função:

[IMG]

#### 3.2.3. Função plugin_init_miniatura() 

É a função central. Também é obrigatória. Ele define a lista de "ganchos" com os quais nosso plugin precisa se conectar.

    function plugin_init_miniatura()
    {
        global $PLUGIN_HOOKS;
        
        $PLUGIN_HOOKS['config_page']['miniatura'] = 'front/plugin_miniatura_config.php';
        $PLUGIN_HOOKS['helpdesk_menu_entry']['miniatura'] = false;
        $PLUGIN_HOOKS['menu_entry']['miniatura'] = false;
        $PLUGIN_HOOKS['item_delete']['miniatura'] = 'plugin_miniatura_item_delete'; 
        $PLUGIN_HOOKS['headings']['miniatura'] = 'plugin_miniatura_get_headings'; 
        $PLUGIN_HOOKS['headings_action']['miniatura'] = 'plugin_miniatura_headings_action';
        
        registerPluginType('miniatura',"PLUGIN_MINIATURA_TYPE",14588, 
            array(
                'classname' => "Plugin_Miniatura",
                'tablename' => "glpi_plugin_miniatura", 
                'typename' => "miniatura",
                'formpage' => NULL,
                'searchpage' => NULL, 
                'template_ables' => false, 
                'deleted_tables' => false, 
                'recursive_type' => false, 
                'specif_entities_table' => false, 
                'reservation_type' => false
            )
        );    
    }

A chamada tem a função registerPluginType() para definir uma nova categoria de objetos a serem gerenciados pelo plugin. Na verdade, vamos armazenar novas informações em uma nova tabela de banco de dados (ver § 3.3). Esta tabela será equiparada a uma classe de objetos.

Os parâmetros são fornecidos: 

+ O nome do plugin (miniatura);
+ Uma string definindo o tipo (PLUGIN_MINIATURA_TYPE);
+ O valor do tipo (ver [GLPI3] para a escolha de valores);
+ Uma matriz associativa:
    + O nome da tabela que armazena informações gerenciadas pelo plugin (tablename);
    + O nome da classe que deriva de commonDBTM e gerencia a tabela de banco de dados (nome da classe);
    + O nome do novo tipo de objeto (nome_do_tipo);
    + Forma de exibição ou de gerenciamento de objetos parâmetros;

#### 3.2.4. Função plugin_miniatura_check_prerequisites()

Esta função obrigatória e não recebe parâmetros. Ela será invocada pelo GLPI para determinar se o plugin pode ser instalado na versão atual do GLPI. Esta função serve para verificar se os pré-requisitos do nosso plugin são atendidos. Aqui, nós apenas verificamos se a versão do GLPI é no mínimo v0.72, e verificamos a versão atual do PHP que tem muitas funções de processamento de imagem que vamos precisar mais tarde:

    function plugin_miniatura_check_prerequisites()
    {
        if (GLPI_VERSION >= 0.72) 
            return true;
            
        echo "A besoin de la version 0.72"; 
        return false;    
    }
    
#### 3.2.5 Função plugin_miniatura_check_config()

Na mesma linha, esta função verifica se a configuração GLPI permite o uso do plugin. No nosso exemplo, nós simplesmente retornamos "true".

    function plugin_miniatura_check_config() 
    {
        return true;
    }
    
#### 3.2.6 Função plugin_miniatura_install()

Antes de explicar o código desta função, é preciso explicar como GLPI determina se um plug-in deve ser instalado ou desinstalado (ver tela anterior no § 3.2.2). Para exibir a lista de plugins disponíveis, GLPI varre o diretório $ROOT/plugins. Para cada arquivo setup.php encontrado, ele chama a função plugin_*_version() e verifica na tabela glpi_plugins se este plugin já está registrado. Em caso caso positivo, verifica também o estado do plugin, em caso negativo, ele adiciona uma nova linha para a tabela de plugins e marca o status do plugin como "Não instalado".

    mysql> select * from glpi_plugins; 
    +----+-----------+----------+---------+-------+-------+---------------------+
    | ID | directory | name     | version | state | author| homepage            | 
    +----+-----------+----------+---------+-------+--------+--------------------+ 
    | 5  | vignette  | Vignette | 1.0     | 2     | jd    | http://www.maje.biz | 
    +----+-----------+----------+---------+-------+-------+---------------------+

A função de configuração deve criar a nova tabela que nos permite armazenar nossas informações complementares:

    function plugin_miniatura_install() 
    {
        $DB = new DB;
        
        $query = "CREATE TABLE `glpi_plugin_miniatura_extinfo` (
            `ID` int(11) NOT NULL PRIMARY KEY AUTO_INCREMENT,
            `id_tracking` int(11) NOT NULL, 
            `miniatura_extticket` char(32) NOT NULL default '', 
            `miniatura_solution` text NOT NULL default ''
        ) ENGINE=MyISAM DEFAULT CHARSET=latin1;"; 
        
        $DB->query($query) or die($DB->error());
        
        return true; // não se esqueça
    }
    
Se a opção "Instalar" for clicada na tela de gerenciamento de plugins, a função anterior é chamada, em seguida, a lista de plugins é reexibida novamente e informa um novo status "Instalado" para o nosso plugin:

[IMG]

O estado também mudou na base de dados:

    mysql> select * from glpi_plugins; 
    +----+-----------+----------+---------+-------+-------+---------------------+ 
    | ID | directory | name     | version | state | author| homepage            | 
    +----+-----------+----------+---------+-------+-------+---------------------+ 
    | 5  | miniatura | Miniatura| 1.0     | 4     | jd    | http://www.maje.biz | 
    +----+-----------+----------+---------+-------+-------+---------------------+

A tela 4 mostra que o plugin ainda deve ser "ativo". Ao selecionar esta opção, GLPI irá executar a função plugin_miniatura_check_prerequisites()  e sucessivamente  plugin_miniatura_check_config() descritas nos tópicos § 3.2.4 e § 3.2.5. Se essas duas funções retornarem verdadeiro, então o plugin será ativado!

[IMG]

De fato, no banco de dados, o seu estado também mudou:

    mysql> select * from glpi_plugins; 
    +----+-----------+----------+---------+-------+-------+---------------------+ 
    | ID | directory | name     | version | state | author| homepage            | 
    +----+-----------+----------+---------+-------+-------+---------------------+ 
    | 5  | miniatura | Miniatura| 1.0     | 1     | jd    | http://www.maje.biz | 
    +----+-----------+----------+---------+-------+-------+---------------------+

#### 3.2.7 Função plugin_miniatura_uninstall()

Nada de especial: A função é chamada quando se seleciona a opção "Desinstalar" na tela de gerenciamento dos plugins. Esta função é o inverso da função de configuração: ela deve apagar a tabela criada: 

    function plugin_miniatura_uninstall()
    {
        $DB = new DB;
        
        $query = "DROP TABLE `glpi_plugin_miniatura_extinfo"; 
        $DB->query($query) or die($DB->error());
        
        return true; // não se esqueça
    }

## 4. Exibindo páginas específicas do nosso plugin

### 4.1. Modelos das páginas exibidas por nosso plugin

Os scripts que criam  as páginas de exibição do conteúdo devem adotar a seguinte estrutura:

+ 1. Definir a constante GLPI_ROOT cujo valor é o caminho raiz do GLPI (nesse exemplo é /opt/glpi);
+ 2. Se você quiser usar objetos do GLPI como TRACKING_TYPE, COMPUTER_TYPE, etc... Você deve então informar isso no array $NEEDED_ITEMS que deve ser incluída antes da inclusão do $ROOT/config/define.php. Isso irá carregar automaticamente os arquivos em que essas classes são declaradas;
+ 3. Mesmo se a interface Web do GLPI exibe apenas os menus e as opções autorizadas privilégios com base em relacionamentos com o perfil do usuário conectado, é recomendável adicionar testes que verifiquem a adequação entre o perfil do usuário logado e ação desejada. Na verdade, um usuário um pouco mais "mal interncionado" do que outros poderia chamar diretamente um script PHP, se ele sabe o seu nome. Para fazer isso, fazemos o uso das funções checkRight() e haveRight(). checkRight() exibe uma mensagem de erro se o perfil não tiver o privilégio, e encerra o script termina enquanto haveRight() retorna apenas um valor booleano, se o usuário tem ou não privilégio;
+ 4. Exibir o menu principal do GLPI é feito chamando a função commonHeader() ;
+ 5. O rodapé é exibido chamando a função commonFooter();
+ 6. As folhas de estilo utilizadas estão situadas em /css/;

O que gera o seguinte esqueleto:

    <?php
        $NEEDED_ITEMS=array(liste d'objets); 
        
        if (!defined('GLPI_ROOT')) {
            define('GLPI_ROOT', '/opt/glpi'); 
        }
        
        include (GLPI_ROOT."/inc/includes.php");
        
        /* verificação das permissões */ /
        //checkRight("config", "w");
        ...
        
        /**
         * Os parâmetros são: 
         *  @param String $title    o título da página, 
         *  @param String $url      a url da página 
         *  @param String $item     Item de exibição, 
         *  @param String $page     o nome da página 
         */        
        commonHeader(lista de parâmetros);
        ...
        commonFooter();
    ?>

### 4.2 Página de configuracão do plugin

Como indicado na função plugin_init_miniatura() do tópico § 3.2.3, o gancho config_page pode dar o nome de um script que será chamado quando você clica no nome do plugin (aqui: Miniatura) em tela 5. Por convenção, quando uma página é construída diretamente por uma ação da interface Web, o script é colocado no diretório. /front do plugin. 
O script ./front/plugin_miniatura_config.php é muito simples, porque nosso plugin não tem parâmetros configuráveis. Não poderíamos definir esse gancho no arquivo setup.php...

    <?php 
        $NEEDED_ITEMS = array("setup"); 
    
        if (!defined('GLPI_ROOT')) {
            define('GLPI_ROOT', '/opt/glpi'); 
        }
    
        include (GLPI_ROOT."/inc/includes.php"); 
        
        checkRight("config", "w");
        
        commonHeader($LANG["common"][12],$_SERVER['PHP_SELF'],"config","plugins"); 
        
        echo "Este plugin não tem parâmetro configurável "; 
        
        CommonFooter();
    ?>
    
Como indicado no tópico § 4.1, sobre os privilégios e função checkRight (), se o perfil do usuário não permite que ele mude a configuração do GLPI, a mensagem de erro é exibida:

Caso contrário, se o perfil permite a configuração GLPI, a seguinte página será exibida:

    Este plugin não tem parâmetro configurável;

### 4.3 Exibição de campos personalizados

Na sua versão atual (0.72), o GLPI não oferece gancho para mudar a tela para introduzir um tiquete. Portanto, se queremos acrescentar novas informações, duas alternativas são possíveis:

+ - Remendar os scripts do GLPI para exibir esses campos;
+ - Gerenciar esses campos específicos em um formulário associado com o plugin;

Por razões de modularidade e para simplificar as atualizações do GLPI, vamos implementar a segunda solução, embora a primeira seja provavelmente, mais ergonômica para o usuário.

#### 4.3.1. Adicionado uma opção do menu Plugins em um tiquete

Para utilizar esta nova forma, precisamos adicionar uma nova guia para o formulário de edição do tiquete:

[IMG]

Esta nova guia "Miniatura" é criada pelo seguinte gancho definido no arquivo setup.php (ver § 3.2.3):

    $PLUGIN_HOOKS['headings']['miniatura'] = 'plugin_miniatura_get_headings';
    
A função plugin_miniatura_get_headings() deve retornar a lista de labels para cada aba que você deseja adicionar na tela de 8. Mas queremos exibir a aba apenas para o formulário de "rastreamento". Para fazer isso, nós filtramos o "tipo" do objeto amarrando-o ao tipo "TRACKING_TYPE":

    function plugin_miniatura_get_headings($type, $withtemplate) 
    {
        // Os tipos de objeto estão definidos no arquivo config/define.php
        witch ($type) {
            case 'TRACKING_TYPE:
                // retorna uma única nova guia
                return array( 1 => "Miniatura" ); 
            break;
        }
        return false;
    }
    
Então nós informamos a ação a ser executada quando a opção "Miniaturas" é selecionada. Também no arquivo setup.php, encontramos a seguinte linha:
    
    $PLUGIN_HOOKS['headings_action']['miniatura'] = 'plugin_miniatura_headings_action';
    
A função plugin_miniatura_headings_action() se parece com a função anterior, mas ao invés de retornar um array de labels, ela retorna o nome das funções a serem chamadas quando o menu é selecionado:

    function plugin_vignette_headings_action($type) 
    {
        switch ($type) { 
            case 'TRACKING_TYPE':
                return array(1 => 'plugin_headings_miniatura');
            break; 
        }
        
        return false;
    }
    
#### 4.3.2. Formulário de gerenciamento dos campos complementares

A função plugin_headings_miniatura() visa complementar o formulário de acompanhamento de um tiquete com os nossos campos complementares! Os seguintes trechos de código mostram o que a função deve produzir: um formulário HTML!

    function plugin_headings_vignette($type, $ID, $withtemplate=0) 
    {
        global $CFG_GLPI, $DB; global $LANG;
        
        // $ID est le numéro du ticket
        if (!$withtemplate || $type != TRACKING_TYPE) 
            return ;
        
        echo "<div align='center'>";
        // Initialise les libellés : $
        title = "Ajouter"; 
        $but_label = "Ajouter"; 
        $but_name = "add"; 
        $solution = "";
        $extticket = ""; 
        $extID = "";
        // Lit les informations actuelles :
        $query = "SELECT * FROM glpi_plugin_vignette_extinfo ".
                 "WHERE id_tracking = '$ID'";
                 
        if ($result = $DB->query($query)){ 
            if ($DB->numrows($result) > 0) {
                $row = $DB->fetch_assoc($result);
                if (!empty($row['vignette_solution'])) {
                    $title = "Modifier";
                    $but_label = "Modifier";
                    $but_name = "modify";
                    $solution = $row['vignette_solution'];                 
                    $extticket = $row['vignette_extticket'];
                    $extID = $row['ID'];                    
                }
            }
        }
        
        echo "<form action='".$CFG_GLPI["root_doc"]."/plugins/miniatura/fron/plugin_miniatura.form.php method='post>\n";
        echo "<input type='hidden' name='id_tracking' value='$ID_tracking'>";
        echo "<input type='hidden' name='ID' value='$extID'>";
        echo "<table class='table_cadre' style='margin: 0; margin-top: 5px;>\n'";
        echo "<tr><th colspan='2'>$title</th></tr>\n";
        echo "<tr class='tab_bg_1'>";
        echo "<td>Ticket externo: </td>";
        echo "<td><input type='text' name='miniatura_extticket' size='20' value='$extticket' ></td>";
        echo "</tr>";
        echo "<tr class='tab_bg_1' >";
        echo "<td>Solução: </td>";
        echo "<td><textarea name='miniatura_solucao' rows='4' cols='100'>$solution</textarea></td>";
        echo "</tr>";
        echo "<tr class='tab_bg_2'>";
        echo "<td colspan='2' align='center' >";
        echo "<input type='submit' name='$but_name' class='submit' value='$but_label' ></td>";
        echo "</tr>\n";
        echo "</table>\n";
        echo "</form>\n";
        echo "</div>";
    }
    
A seleção da aba "Miniatura" no formulário de acompanhamento do tiquete, faz com que a nossa função seja chamada e nosso novo formulário é exibido como mostrado abaixo:

[IMG]

Podemos acrescentar novas informações ao pressionar o botão "Adicionar" que chama script $ROOT/plugins/miniatura/ fron/plugin_miniatura.form.php, cuja função é armazenar as novas informações no banco de dados. A classe plugin_Miniatura herda a classe CommonDBTM que irá nos fornecer métodos que serão de grande ajuda.

O código do plugin_miniatura.form.php é o seguinte:

    <?php
        define('GLPI_ROOT', '/opt/glpi');
        
        include_once (GLPI_ROOT."/inc/includes.php"); 
        include_once "../inc/plugin_miniatura.classes.php";
        
        checkRight('config',"w");
        $vignette = new plugin_Miniatura(); 
        
        if (isset($_POST["add"])) {
            $newID = $vignette->add($_POST); 
            glpi_header($_SERVER['HTTP_REFERER']);
            
        } elseif (isset($_POST["modify"])) {
            $newID = $vignette->update($_POST); 
            glpi_header($_SERVER['HTTP_REFERER']);
            
        } else {
            glpi_header("../index.php");
        }
    ?>

[IMG]

Sim! percebemos que a partir de agora existem informações relacionadas com o tiquete. Isto também mostra o seguinte conteúdo da base de dados:

    mysql> select * from glpi_plugin_miniatura_extinfo; 
    +----+-------------+--------------------+-------------------------------+ 
    | ID | id_tracking | miniatura_extticket| miniatura_solution            | 
    +----+-------------+--------------------+-------------------------------+ 
    | 1  | 22          | ABCD1234           | Solução de teste              |
    +----+-------------+--------------------+-------------------------------+ 
    1 row in set (0.00 sec)
    
#### 4.3.3. Exibição de Miniaturas e Wallpaper

Como mencionamos na introdução, nós também desejamos anexar um tiquete de print-screens da tela. O GLPI te permite, vincular qualquer tipo de arquivo com um tiquete. No entanto, GLPI não oferece pré-visualização se esses arquivos são imagens.

A mudança a ser feita é bastante simples: alterar a função plugin_headings_miniatura() do tópico § 4.3.2, a fim de mostrar as miniaturas das imagens associadas com tiquetes.

Sabendo-se que as associações entre tiquetes e arquivos são descritas na tabela glpi_docs e que os arquivos são armazenados no diretório $ROOT/files, basta adicionar o seguinte fragmento de código:

    function plugin_headings_miniatura(...)
    {
        ...
        $query = "SELECT * FROM glpi_docs WHERE FK_tracking = $ID_tracking"; 
        if ($result = $DB->query($query)){
            if ($DB->numrows($result) > 0) {
                echo "<table class=\"tab_cadre\" style=\"margin: 0; margin-top: 5px;\">\n";
                echo "<tr><th colspan=\"2\">Images associ&eacute;es</th></tr>\n";
                echo "<tr><td>\n";
                
                while ($row = $DB->fetch_assoc($result)) {
                    # Tester les PNG, GIF, PJG
                    $fname = GLPI_ROOT."/files/".$row['filename']; 
                    if (@getimagesize($fname) === FALSE) continue; 
                    $type = dirname($row['filename']);
                    echo "<a target=\"_blank\" href=\"$fname\">".
                    "<img src=\"".$CFG_GLPI['root_doc']. "/plugins/vignette/front/plugin_vignette_thumbnail.php?image="urlencode(basename($fname)) &type=$type\" width=100 height=80></a> \n";
                }
                
                echo "</td></tr>\n"; echo "</table>\n";        
            }
        }
        ...
    }
    
O código de script Plugin_miniatura_thumbnail.php, que constrói miniaturas "em tempo de execução", está disponível no site [MAJE1].

A tela abaixo mostra a exibição do tiquete que teve adicionadas duas imagens PNG:

[IMG]

E clicar em uma miniatura, ela será exibida em uma nova janela em tamanho completo.

### 4.4. Caso de destruição de tiquetes - gancho "delete"

Se você implementou o código proposto até agora, ou se você simplesmente tem entendido o princípio do nosso plugin, você pode ver que é ele capaz de anexar informações complementares e miniaturas a um tiquete.

Mas o que acontece em caso da exclusão de um tiquete? Bem, o GLPI não "lembra" que nós anexamos informações complementares nesse tiquete - além do mais, se você destruir um tiquete, você vai perceber que os anexos não são apagados (por opção de implementação sem sem dúvida)!

Cabe a nós para limpar a tabela glpi_plugin_miniatura_extinfo quando um tiquete é excluído. É aqui que o gancho item_delete nos trás toda a sua utilidade. No arquivo setup.php, definimos a seguinte função, que será chamada com como parâmetro, o ID do ticket excluído. Então precisamos recuperar o ID da informação complementar em nossa tabela porque a função deleteFromDB() da classe CommonDBTM sempre excluí um registro usando uma coluna de identificação como critério. O que nos dá:

    function plugin_miniatura_item_delete($parm)
    {
    
        if (!isset($parm['ID'])) 
            return true;
        
        switch ($parm['type']) {
            case TRACKING_TYPE:            
                if ($vid = $db->getVIDFromTID($parm['ID']))
                    $db->deleteFromDB($vid);
            break; 
        }
        
        return true;
    }
    
E então nós adicionamos função um getVIDFromTID() no arquivo inc/plugin_miniatura.classes.php:

    public function getVIDFromTID($tid)
    {
        global $DB;
        
        $query = "SELECT * FROM ".$this->table." WHERE id_tracking = $tid"; 
        
        if ($result = $DB->query($query)) {
            if ($DB->numrows($result) == 1) {
                $fields = $DB->fetch_assoc($result); 
                return $fields['ID'];
            } 
        }
        
        return NULL;
    }
    
## 5. Outras extensões possíveis

### 5.1 Formulário de pesquisa

O GLPI oferece ganchos para os plugins oferecem seu próprio formulário de busca. 
Primeiro, temos que escrever a função plugin_miniatura_getSearchOption(), que irá listar os campos que serão visualizados no formulário de busca.

Cada campo é descrito por: 
+ - O nome da tabela onde está armazenado (atributo 'table'); 
+ - O nome do campo na tabela(atributo 'field');
+ - O nome de um campo que é uma chave estrangeira (campo 'Linkfield');
+ - Legenda para exibir (atributo 'name');

Em seguida, devemos invocar a função searchForm() que exibe o formulário. Uma vez que a seleção feita pelo usuário, a função showList() é chamada. A construção da consulta SQL é um dependente.

Como o algoritmo de busca é baseado em uma consulta SQL, o GLPI pode definir as funções plugin_miniatura_addLeftJoin(), plugin_miniatura_forceGroupBy(), plugin_miniatura_addWhere(), plugin_miniatura_addHaving(), plugin_miniatura_addSelect() e plugin_miniatura_addOrderBy() para criar a consulta SQL que corresponda exatamente às nossas necessidades.

### 5.2 Exportação de dados

Nossa tabela glpi_plugin_miniatura_extinfo é salva automaticamente quando o GLPI exportar dados através do menu (Central> Administração> Dados) formato SQL ou XML. O GLPI pode gerar relatórios dinâmicos.

### 5.3 Outras possibilidades

+ - Mudanças "em massa": na língua do GLPI é a possibilidade de aplicar uma ação há um conjunto de objetos selecionados em uma lista;
+ - A integração com o cronograma de intervenção; 
+ - Gerenciar os campos de entrada do tipo "lista de seleção" para o GLPI, é um campo "lista suspensa" e o GLPI prontamente permite a ligação entre o conteúdo de uma tabela e uma entrada da lista.

Essas outras funcionalidades podem ser um segundo item, se o primeiro item já foi tratado...

# Conclusão 

Neste artigo, tentamos esclarecer as informações disponíveis no site [GLPI1], mostrando as com um caso concreto. Outra boa maneira de desenvolver o seu próprio plugin é estudar alguns dos muitos plugins já desenvolvidos e propostos no site do GLPI. 

Como API GLPI é muito rica (ver referência [GLPI2]), também recomendamos que você estudar o conteúdo do diretório ./inc. Ele é cheio de funções e classes que permitem o acesso a objetos no GLPI e oferecem tratamentos comuns. 

Finalmente, não se esqueça de colocar o GLPI no modo "debug" no menu (Central > Configurações> Geral), os seus erros do PHP serão exibidos e os valores das variáveis $_SESSION, $_POST e $_GET, para não mencionar as consultas SQL .... isso é muito útil!
