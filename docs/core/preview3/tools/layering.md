---
title: "Arquitetura das Ferramentas da Linha de Comando da Visualização 3 do .NET Core"
description: "A Visualização 3 traz determinadas alterações na forma como as ferramentas gerais do .NET Core são colocadas em camadas."
keywords: CLI, extensibilidade, comandos personalizados, .NET Core
author: blackdwarf
manager: wpickett
ms.date: 11/12/2016
ms.topic: article
ms.prod: .net-core
ms.technology: .net-core-technologies
ms.devlang: dotnet
translationtype: Human Translation
ms.sourcegitcommit: 1a84c694945fe0c77468eb77274ab46618bccae6
ms.openlocfilehash: 233d365b582c274cd3a1f078846a6e854c7a6c95

---

<a name="high-level-overview-of-changes-in-cli-preview-3"></a>Visão geral de nível alto das alterações na CLI da Visualização 3
-----------------------------------------------

# <a name="overview"></a>Visão Geral
Este documento descreverá em alto nível as alterações causadas pela mudança de `project.json` para o MSBuild e o sistema de projeto `csproj`. Ele descreverá a nova maneira pela qual as ferramentas são todas colocadas em camadas e quais peças novas estão disponíveis, bem como seus lugares no quadro geral. Após ler este artigo, você terá uma compreensão melhor sobre todas as partes que compõem as ferramentas do .NET Core após a mudança para o MSBuild e `csproj`. 

> **Observação:** este artigo é **não é necessário** para usar as ferramentas de Linha de Comando da Visualização 3 do .NET Core. Você pode continuar usando as ferramentas às quais está acostumado. Este artigo está completa o quadro de como a mudança para o MSBuild altera a "camada" geral e a arquitetura das ferramentas de linha de comando. 

# <a name="moving-away-from-projectjson"></a>Deixando o project.json
A maior alteração na ferramentas da Visualização 3 do .NET Core é certamente a [mudança do project.json para o csproj](https://blogs.msdn.microsoft.com/dotnet/2016/05/23/changes-to-project-json/) como sistema de projeto. A versão de Visualização 3 das ferramentas de linha de comando é a primeira versão das ferramentas de linha de comando do .NET Core que não oferece suporte ao project.json. Isso significa que ele não pode ser usado para compilar, executar ou publicar aplicativos e bibliotecas baseados no project.json. Para usar essa versão das ferramentas, é necessário migrar os projetos existentes ou iniciar projetos novos. 

Como parte dessa mudança, o mecanismo de build personalizado desenvolvido para compilar projetos project.json foi substituído por um mecanismo de build maduro e totalmente capaz, chamado [MSBuild](https://github.com/Microsoft/msbuild). O MSBuild é um mecanismo conhecido na comunidade do .NET, pois que tem sido uma tecnologia chave desde a primeira versão da plataforma. Naturalmente, como precisa compilar aplicativos .NET Core, o MSBuild foi movido para o .NET Core e pode ser usado em qualquer plataforma que executa o .NET Core. Uma das principais promessas do .NET Core é uma pilha de desenvolvimento de plataforma cruzada e nós garantimos que essa mudança não quebra essa promessa.

> **Observação:** se você for novato no MSBuild e deseja de saber mais sobre ele, comece lendo a [documentação existente](https://msdn.microsoft.com/en-us/library/dd637714.aspx). 

# <a name="the-tooling-layers"></a>As camadas de ferramentas
Com a mudança do sistema de projeto existente, bem como com a criação de comutadores de mecanismo, a pergunta que surge naturalmente é: essas mudanças alteram as "camadas" gerais do ecossistema de ferramentas do .NET Core? Existem novos bits e componentes?

Vamos começar recapitulando as camadas na Visualização 2, conforme mostrado na figura a seguir:

![A arquitetura de alto nível das ferramentas da Visualização 2](media/p2-arch.png)

As camadas das ferramentas são bastante simples. Na parte inferior, temos as ferramentas de Linha de Comando do .NET Core como base. Todas as outras ferramentas de alto nível, como o Visual Studio ou o código VS, dependem e contam com a CLI para compilar projetos, restaurar dependências e assim por diante. Isso significa que, por exemplo, se o Visual Studio quisesse executar uma operação de restauração, ele chamaria o comando `dotnet restore` na CLI. 

Com a mudança para o novo sistema de projeto, o diagrama anterior se altera: 

![A arquitetura de alto nível das ferramentas da Visualização 3](media/p3-arch.png)

A principal diferença é que a CLI não é mais a camada básica; agora, essa função é executada pelo "componente SDK compartilhado". Esse componente SDK compartilhado é um conjunto de destinos e tarefas associadas responsáveis por compilar o seu código, publicá-lo, empacotar pacotes nuget etc. O SDK em si é aberto e está disponível no GitHub no [repositório SDK](https://github.com/dotnet/sdk). 

> **Observação:** "destino" é um termo do MSBuild que indica uma operação nomeada que o MSBuild pode invocar. Normalmente, ele está acoplado com uma ou mais tarefas que executam alguma lógica que o destino deve realizar. O MSBuild oferece suporte a muitos destinos prontos como `Copy` ou `Execute`; ele também permite aos usuários gravar suas próprias tarefas usando código gerenciado e definir metas para executar essas tarefas. Você pode ler mais sobre as tarefas do MSBuild em [MSDN](https://msdn.microsoft.com/en-us/library/ms171466.aspx). 

Todos os conjuntos de ferramentas agora consumem o componente SDK compartilhado e seus destinos, incluindo a CLI. Por exemplo, a próxima versão do Visual Studio não chamará o comando `dotnet restore` para restaurar dependências para projetos do .NET Core, ela usará o destino de "Restauração" diretamente. Como esses são os destinos do MSBuild, também é possível usar o MSBuild bruto para executá-los, usando o comando [dotnet msbuild](dotnet-msbuild.md). 

## <a name="preview-3-cli-commands"></a>Comandos CLI da Visualização 3
O componente SDK compartilhado significa que a maioria dos comandos CLI existentes foram reimplementados como destinos e tarefas do MSBuild. O que isso significa para os comandos CLI e o uso do conjunto de ferramentas? 

De uma perspectiva de uso, ele não altera a maneira de usar a CLI. A CLI ainda tem os comandos principais que existem na versão da Visualização 2:

* `new`
* `restore`
* `run` 
* `build`
* `publish`
* `test`
* `pack` 

Esses comandos ainda fazem o que você espera que eles façam (criar um projeto, compilá-lo, publicá-lo, empacotá-lo e assim por diante). A maioria das opções não foram alteradas e ainda estão lá. Você pode consultar as telas de ajuda dos comandos (usando `dotent <command> --help`) ou documentação da Visualização 3 neste site para se familiarizar com as alterações. 

De uma perspectiva de execução, os comandos CLI assumirão seus parâmetros e construirão uma chamada para o MSBuild "bruto" que defina as propriedades necessárias e execute o destino desejado. Para ilustrar melhor, considere o seguinte comando: 

    `dotnet publish -o pub -c Release`. 
    
Esse comando está publicando um aplicativo em uma pasta `pub` usando a configuração de "Versão". Internamente, esse comando é convertido na seguinte invocação do MSBuild: 

    `dotnet msbuild /t:Publish /p:OutputPath=pub /p:Configuration`

A exceção notável a essa regra são os comandos `new` e `run`, pois eles não foram implementados como destinos do MSBuild. 

# <a name="conclusion"></a>Conclusão 
Este documento descreveu em alto nível as alterações que estão ocorrendo na arquitetura e no funcionamento geral das ferramentas da CLI que serão apresentadas com a Visualização 3. Introduziu-se a noção de componente SDK compartilhado se explicou como os comandos da CLI funcionam, de uma perspectiva técnica, na Visualização 3. 




<!--HONumber=Nov16_HO3-->


