Injeção de dependência usando RequireJS e AngularJS
===================================================

Como a injeção de dependência (DI) do RequireJS é usada com a DI do AngularJS?

Desenvolvedores usam RequireJS para assincronamente carregar Javascript e estabelecer dependências de importação/pacotes. E o framework AngularJS é usado para construir HTML SPAs(single-page applications) usando injeção de dependência (DI) e ligação de dados (databinds). Desenvolvedores trabalhando em aplicações HTML5 não triviais rapidamente percebem a imporância de usar RequireJS com AngularJS. Infelizmente, muitos desses desenvolvedores facilmente se tornam confusos quando eles percebem que ambos os frameworks supostamente realizam injeção de dependência (DI).

E se ambos, AngularJS e RequireJS suportam DI, então os desenvolvedores rapidamente começam a se perguntar:

1. Como RequireJS DI é usado com AngularJS?
2. A injeção de dependência do RequireJS é a mesma injeção de dependências do AngularJS?
3. Como RequireJS injeta valores usados para preparar instâncias injetadas pelo AngularJS?

Para responder a essas questões, vamos primeiro explorar como objetos/classes javascript podem ser empacotadas em arquivos discretos e gerenciáveis.

O padrão de projetos Module Pattern
-----------------------------------

O padrão de projetos Module Pattern é um padrão de codificação javascript amplamente usado quando os desenvolvedores querem arquivos/módulos javascript discretos:

    //
    (function() {
    
    // ...
    
    }());
    //

### Importações globais

Todo o código que é executado dentro da função  vive em uma clousure, que fornece privacidade e estado em todo o ciclo de vida da nossa aplicação. Para evitar o uso de variáveis globais implicitas, o padrão de projeto Module Pattern pode especificar parâmetros para a função anônima.

    (function (define, angular) {
    
    // ...
    
    }(define, angular));

Ao passar variáveis globais como parâmetros para a nossa função anônima, nós estamos importando essas variáveis em nosso código, que é ao mesmo tempo mais claro e mais rápido do que variáveis globais implicitas.

> Infelizmente, a máquina V8 atualmente apresenta bugs de breakpoints com Module Pattern e parâmetros. Sendo assim, parâmetros não são atualmente recomendados para propósitos de desenvolvimento. Em adição a isso, desde que os parâmetros nós devemos passar são de fato globais (define, angular), embora seja claro não é necessário fazer isso.

### Exportação de módulos

Algumas vezes você não quer apenas usar globais, mas você também deseja declará-los. Nós podemos facilmente fazer isso ao exportá-los, usando o retorno da função anônima.

    window.utils = (function()
    {
       // publica uma função utilitária 'log()'
       return {
          log: function(message){
             console.log(message);
          }
       };
          
    }());
    
O exemplo acima nos mostra como funções anônimas publicam um objeto 'utils' com uma função 'log()' no escopo global.

> Exportação de módulos não são usados dentro do nosso projeto, visto que abordagem requer atribuição explícita de variáveis ou locais de armazenamento. Ao invés disso, nós usamos o padrão AMD para gerenciar a exportação de módulos e dependências.

Definição do padrão de projeto AMD (Assynchronous Module Pattern)
-----------------------------------------------------------------

O padrão AMD permite ao desenvolvedor:

+ Definir módulos JavaScripts discretos... módulos que podem ter 1..n dependências de outros módulos, e
+ Carregar essas dependências assíncronamente (e.g. em algum momento bem depois)

Usando o padrão AMD do RequireJS, vamos dar uma olhada em como definir uma classe Logger:

    /**
     * Log fornece um método abstrato para encapsular mensagens de log no console
     * Nome do arquivo: "myApp/utils/logger.js"
     */
    var dependencias = []; // lista de nomes de arquivos/módulos JS a serem carregados primeiro
    
    define (dependencias, function()
    {
       // Publica uma função utilitária "log()"
       var logger = {
          log: function(message) {console.log(message);},
          debug: function(message) {console.debug(message);};
       };
       
       return logger;
    });

É importante perceber que o _define()_ não retorna um valor. Pelo contrário, a função anônima especificada dentro _define()_ retorna um valor para RequireJS ... um valor que é registrado/armazenado em cache para  uso/injecção posterior.

Desenvolvedores podem pensar o seguinte sobre o código acima:

+ Verifica se as dependências estão carregadas (neste caso, nenhuma);
+ Executa a função anônima (usualmente feito assincronamente depois que todos os _defines()_ foram carregados e suas árvores de dependências foram construídas;
+ Armazena o valor retornado em um registro de definição associado com a chave myApp/utils/logger;
+ Injeta o valor armazenado em funções que declaram _logger_ como uma dependência;

Lembre-se que o padrão AMD precisa retornar um valor... um valor que será armazenado no registro dedefinição [para futuras injeções]. A coisa mais importante a se observar é que o valor pode ser *qualquer coisa*: objeto, string, array, ou mesmo uma função.

O tipo específico de valor retornado é inteiramente dependente de como ou o que os desenvolvedores querem injetá-lo em suas funções dependentes.

AMD do RequireJS e Injeção de dependência
-----------------------------------------

Em adição ao gerenciamento de importação de dependências e registro de valores de retorno, o padrão AMD tem um (1) outro recurso muito importante: **Injeção de Dependência**.
Quando a função utilitária _logger_ é necessária dentro de outro módulo, você simplesmente _define_ uma dependência para carregar o módulo _logger_ e o RequireJS irá **injetar** uma referência para o valor _logger_ no registro de definição AMD.

> A definição é ditamente carregada quando a função anônima [com seus parâmetros] é invocada e o valor é retornado.

    /**
     * Greeter
     * Nome do arquivo: "myApp/Greeter.js"
     */
    define( ['myApp/utils/logger'], function(logger)
    {
        // publica a instância do Greeter
        return {
            welcomeUser: function(user){
                logger.debug("Bem vindo: " + user);
            }
        }
    });
    
Observe que o argumento _logger_ na função anônima acima irá agora referenciar a instância _logger_ que está armazenada no registro de definição... e este valor é gerado como o valor de retorno da função anônima no próprio módulo _myApp/utils/logger. 
Uma vez que a corrente de dependência especifica que a classe Greeter não pode ser definida até que a definição _logger_ tenha sido carregada, isso efetivamente significa que o RequireJS **injetou** o valor logger na classe Greeter (durante a instanciação em tempo de execução).

### Uso de AMD complemente protegidos

Finalmente, vamos empacotar o padrão AMD do RequireJS dentro do padrão Module:

    (function()
    {
        var numGreetings = 0;
        var dependencias = [
            'myApp/utils/logger' // diz ao RequireJS para injetar o valor retornado de Logger.js
        ];
        
        define(dependencies, function(logger)
        {
            // publica a intância do Greeter
            return {
                welcomeUser : function(user)
                {
                    numGreetings += 1;
                    logger.debug("Seja bem vindo " + user);                    
                },
                reportGreetings : function()
                {
                    logger.debug("Número de greetings emitidos = " + numGreetings);
                }                
            };
        });
    }());

O uso do padrão Module (acima) permite ao desenvolvedor:
+ Proteger as partes internas de como a classe Greeter é definida;
+ Encapsular e gerenciar o estado global privado numGreetings;

> Esta abordagem de definir um padrão AMD do RequireJS dentro de um módulo é usada dentro de cada classe em meus projetos HTML5.

Desenvolvedores devem observar, entretanto, que a injeção de dependência AMD do RequireJS (DI) não é um processo de injeção do AngularJS. O DI do RequireJS é usado apenas para injetar funcionalidade que está em um arquivo externo e separado de qualquer construções de instâncias do AngularJS.

Em seções subsequentes, nós iremos ver como a DI do Angular difere e como ambos os DI podem ser usados juntos em soluções sinergéticas.

### Injeção de dependência do AngularJS

AngularJS fornece um suporte único e poderoso para injeção de dependência que é distinto das injeções de dependência AMD do RequireJS.

> Pense na DI do angular como injeção de instâncias e na DI do RequireJS  como injeção de classes ou construtores.

Assim como o RequireJS, AngularJS fornece mecanismos para construir um registro de instâncias gerenciados por palavras chave: _factory()_, _service()_ e _controller()_. Todas as três (3) funções recebem:

+ chave: considere ela como o nome da instância da variável;
+ função: considere ela como a função de construção;

    angular
        .factory("session", function()
        {
            return Session();
        })
        .service("authenticator", function()
        {
            return new Authentication();
        })
        .controller("loginController", function(session, authenticator)
        {
            return new LoginController(session, authenticator);
        });

Usar uma função de construção, permite ao AngularJS internamente adiar a invocação até que a instância seja realmente solicitada.

Mas observou como a função de construção para o _loginController_ requer instâncias de _session_ e _authenticator_ como parâmetros da função? Essas instâncias são injetadas na invocação da função quando o AngularJS instancia uma instância de LoginController.

#### Usando anotações AngularJS para a injeção

Vamos reconsiderar o registro do módulo /myApp/controllers/LoginController.js como um controlador no AngularJS:

    (function()
    {
        define( function()
        {
            var LoginController = function(session, authenticator)
            {
                // publica uma instância com APIs simples de login() e logout()
                return {
                    login: function(userName, password)
                    {
                        authenticator.login(userName, password);
                    },
                    logout: function()
                    {
                        authenticator.logout().then(function()
                        {
                            session.clear();
                        });
                    }
                };
            };
            
            // publica a função construtora
            return LoginController;
        });
    }());

Aqui a função construtora publicada esperada... onde a referência LoginController é realmente uma Função.

Como eu mencionei anteriormente, AngularJS injeta valores de parâmetros como argumentos para o construtor. Mas como o AngularJS conhece quais instâncias/valores injetar? JavaScript não tem anotações, e anotações são necessárias para a injeção de dependência.

Bem, você tem que dizer ao AngularJS os nomes das instâncias a serem injetadas. Todas as maneiras a seguir são maneiras válidas de anotar funções com injeção de argumentos e são equivalentes:

+ Infererido: Use o método Function.toString() para parsear e determinar o nome dos argumentos. Observe, que isso apenas funciona se o código não precisar ser minificado/obfuscado;
+ Anotado: Adicionando uma propriedade $inject na função onde os parâmetros são especificados como elementos em um array;
+ Inline: como um array de nomes de injeção, onde o último item do array é a função a ser chamada;

> Ambas as técnicas Anotado e Inline também continuam a funcionar da forma esperada mesmo quando nós utilizamos minificação para obfuscar e comprimir nosso código da aplicação em produção. E para desenvolvedores preocupados quando à performance, a máquina AngularJS realmente scaneia o método de anotação na seguinte ordem de prioridade: $inject, inline, inferido.

O suporte inline do AngularJS é uma técnica elegante onde os desenvolvedores podem usr um array onde cada elemento do array é uma string cujo nomes correspondem a um argumento da função a ser invocada E o último elemento é o próprio construtor da função. Eu pessoalmente chamo esta técnica de _array construtor_.

Eu vejo o suporte embutido para cada função construtora ou um array como uma solução incrivelmente engenhosa e poderosa. Aqui está como eu uso essa técnica inline, _array construtor_ como parte de uma configuração do AngularJS:

    angular
        .controller("LoginController", ["session", "authenticator", function(session, authenticator)
        {
            return new LoginController(session, authenticator);
        }]);

Isso funciona... mas os desenvolvedores devem imediatamente observar 2 considerações nesta solução:

1. O registro do AngularJS do código do "LoginController" é muito confuso usando a **inline**, array construtor.
2. Se a implementação do LoginController mudar e o argumentos do construtor forem diferentes, desenvolvedores precisam se lembrar de também modificar/atualizar o array construtor acima.

> Como mencionado anteriormente, Angular suporta a técnica de anotação $inject (que também possibilita a minificação de código). Ao invés de usar arrays construtores, desenvolvedores podem anexar uma propriedade $inject à função construtora. O valor desta propriedade é um array de nomes dos parâmetros usados como argumentos:
>   LoginController.$inject = ["session", "authenticator"]
> Observe que esta técnica foi depreciada, uma vez que nós queremos usar **inline**, Array construtor com RequireJS.

#### Usando Arrays construtores dentro do AngularJS

Nós podemos facilmente refatorar nosso código para ao mesmo tempo resolver as duas questões acima e suportar a minificação e anotação.. usando **inline**, arrays construtores ao invés de funções construtoras. O truque é ter RequireJS injetando o **inline**, array construtor dentro do módulo onde nós configuramos o AngularJS.

O código abaixo irá demonstrar. Primeiro, vamos atualizar o módulo LoginController para publicar um ** array construtor, inline** ao invés de uma função:

    (function()
    {
        define(function()
        {
            var LoginController = function(session, authenticator)
            {
                // publica uma instância com APIs simples login() e logout()
                return {
                    login: function(username, password) {/***/},
                    logout: function(){ /***/ }
                };
            }
            
            // publica o array construtor/construção 
            return ["session", "authenticator", LoginController];
        });
    }());

Agora nosso código de configuração da inicialização do AngularJS está grandemente simplificado:

    (function()
    {
        var dependencies = [
            "myApp/model/Session",
            "myApp/services/Authenticator",
            "myApp/controllers/LoginController"
        ];
        
        define (dependencies, function(Session, Authenticator, LoginController)
        {
            angular
                .factory("session", Session)
                .service("authenticator", Authenticator)
                .controller("loginController", LoginController);
        });
    }());
    
Desenvolvedores devem observar muitos aspectos interessantes no código acima.

Aqui nós temos alavancado o poder dos valores de retorno do AMD do RequireJS que são injetados como parâmetros para a função anônima _define()_. Esses parâmetros se parecem como referências de classes... mas são realmente arrays construtores do AngularJS. E o resultado do código de configuração acima é significativamente mais fácil de manter. De fato, agora o código de configuração não indica nada sobre anotações ou minificação segura ou especificações de nomes de argumento... Apenas funciona!

Os parâmetros AMD injetados são então usados dentro do AngularJS para suportar a criação dinâmica e subsequentemente injeção de dependência. AngularJS irá detectar que o array construtor está especificado, determinar os valores apropriados do DI que precisam ser injetados como argumentos do construtor, e então invocar a função construtora. E finalmente, o valor retornado pela função construtora é armazenado dentro de um registro do Angular; pelo seu nome de variável.

> Eu preciso enfatizar que quando a função construtora é invocada, o valor publicado ou 'retornado' PRECISA ainda ser um valor apropriado ou instância de objeto. Este valor/instância será cacheado pelo AngularJS para subsequentes e futuras injeções.

Observe que a chave de registro [usada no exemplo de código acima] _session_, _authenticator_, e _loginController_ são lowercase porque seus respectivos valores são **instâncias** criadas através da invocação de suas funções construtoras. Em contraste, nomes upper camel case indicam tipos que são realmente, **classes**, **construtores** ou **funções**... referências dos tipos que foram injetados pelo RequireJS.

#### Considerações de Injeção AMD

Como observado acima, na maioria dos casos a dependência AMD do RequireJS nos permite injetar classes ou "arrays construtores". Algumas vezes, também é necessário que o RequireJS também injete uma função que não será utilizada/consumida pelo AngularJS.

Como um exemplo, vamos considerar uma função  "supplant" que fornece recursos para construir strings completas com parâmetros de acordo com tokens. Em nosso exemplo abaixo, nós queremos construir strings e fazer o log de mensagens independentemente dos mecanismos e processos do $log do AngularJS. Assim, vamos ibjetar a função "supplant" [publicada no módulo myApp/utils/supplant.js]... junto com nossas outras classes _Session_, _Authenticator_, _LoginController_: 

    (function()
    {
        var dependencies = [
            'myApp/utils/supplant',
            'myApp/model/Session',
            'myApp/services/Authenticator',
            'myApp/controllers/LoginController'
        ]
        
        define(dependencies, function(supplant, Session, Authenticator, LoginController)
        {
            console.log(supplant(
                "Adicionada dependência 'supplant()' do módulo {0}",
                dependencies
            ));
            
            angular
                .factory("session", Session)
                .service("authenticator", Authenticator)
                .controller("loginController", LoginController);
        });
    }());
    
Aqui a função "supplant" é usada apenas com o console.log e não é usado com o AngularJS.

Este simples exemplo, mostra como é fácil usar RequireJS para injetar ambas as funções (a) de uso geral e (b) Classes (funções construtoras ou arrays construtores) para instanciação do AngularJS e para usos do DI.

### Padrões de codificação de módulos recomendadas

Uma condensação desta inteira exploração pode ser sumarizada da seguinte forma:

> Considere que DI do AngularJS injeta instâncias, e o DI AMD do RequireJS injeta classes ou construtores.

Ao definir classes e pacotes, eu sugiro que as seguintes heurísticas sejam consideradas como padrões de desenvolvimento HTML5.

+ Todas as classes/módulos devem ser empacotadas em um módulo raiz, sem qualquer importação global;
+ Use AMD RequireJS para estabelecer/definir dependências de módulos e ordens de invocação;
+ Use AMD RequireJS para injetar valores de módulos (Classes, funções, arrays construtores);
+ Se os valores do Módulo são usados dentro do AngularJS, então esses módulos devem publicar um valor para o array construtor.
