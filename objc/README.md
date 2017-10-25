## Lista de providers disponíveis

- [STNPinPadConnectionProvider](#criação-de-sessão-com-o-pinpad) - Estabelece sessão entre o aplicativo e o pinpad

- [STNStoneCodeActivationProvider](#ativação-do-stone-code) - Ativa o Stone Code do lojista

- [STNTableDownloaderProvider](#download-das-tabelas-aid-e-capk) - Baixa tabelas AID e CAPK do servidor para o celular

- [STNTableLoaderProvider](#carregamento-das-tabelas-aid-e-capk-para-o-pinpad) - Carrega as tabelas AID e CAPK para o pinpad

- [STNTransactionProvider](#envio-de-transações) - Captura o cartão do comprador e envia a transação

- [STNTransactionListProvider](#listagem-de-transações)  - Lista transações com opção de listar pelo cartão do comprador

- [STNMerchantListProvider](#listagem-de-lojistas)  - Lista lojistas ativados no aplicativo

- [STNCancellationProvider](#cancelamento-de-transações) - Cancelamento de transações

- [STNMailProvider](#envio-de-comprovante-por-email) - Envia email com comprovante da transação ou cancelamento

- [STNValidationProvider](#validações) - Responsavel pelas seguintes validações: se há conexão com a internet, se o Stone Code está ativado, se há conexão com o pinpad e se as tabelas já foram baixadas

- [STNCardProvider](#captura-de-pan) - Captura os 4 últimos números do cartão

- [STNDisplayProvider](#exibição-no-display-do-pinpad) - Exibe mensagem de até 32 caracteres no vizor do pinpad

## Lista de models disponíveis

- [STNTransactionModel](#transação) - Model com propriedades da transação

- [STNMerchantModel](#lojista) - Model com propriedades do lojista

- [STNPinpadModel](#pinpad) - Model com propriedades do pinpad

- [STNAddressModel](#endereço) - Model com propriedades de endereço do lojista

## Outras classes disponíveis

- [STNConfig](#configurações) - Configurações gerais

- [STNPinpad]() - Objeto que representa o pinpad

## Utilização

### Importação das classes do SDK

Para acessar todas as classes do SDK basta importar o StoneSDK.

```objective-c
#import <StoneSDK/StoneSDK.h>
```

### Configurações

Para customizar as mensagens mostradas na tela do pinpad, passe um `NSDictionary` contendo os valores requeridos com chave e valor do tipo `{STNTransactionMessage: @"NSString"}`.
Por exemplo, o valor a seguir irá substituir a mensagem para Transaction Declined:
```objective-c
  @{@(STNTransactionMessageDeclined) : @"TRANSACAO       XYZ"}
```
> Lembre-se que o display pode mostrar apenas 32 digitos, sendo 16 para cada linha.

Para trocar entre chaves Elavon e Stone, deve-se definir o STNAcquirer no STNConfig.
```objective-c
  [STNConfig setAcquirer:STNAcquirerStone];
```


### Criação de sessão com o pinpad Bluetooth (dispositivo MFi)

Para realizar qualquer comunicação com o pinpad é necessario que se crie uma sessão. Lembrando que a conexão com o dispositivo bluetooth deve ser feita no menu de *Ajustes* do *iOS*.

> Antes de qualquer comunicação entre o aplicativo e o pinpad, uma sessão deve ser estabelecida.

```objective-c
    [STNPinPadConnectionProvider connectToPinpad:^(BOOL succeeded, NSError *error)
    {
        if (succeeded) // verifica se a requisição ocorreu com sucesso
        {
            // executa alguma coisa
        } else
        {
            // trata o erro
            NSLog(@"%@", error.description);
        }
    }];
```

> Recomendamos que esse método seja executado a cada **3 minutos** quando o aplicativo estiver em background caso usem o pinpad `Gertec MOBI PIN 10`. O dispositivo em questão apresentou problemas ao ficar muito tempo sem comunicação.

#### Possivel código de erro

[303](#códigos-de-erro)

### Conexão com pinpad Bluetooth Low Energy (BLE)

Para conectar-se com um pinpad Bluetooth Low Energy, deve-se primeiro implementar o delegate `STNPinPadConnectionDelegate` e inicializar  o central manager presente no `STNPinpadConnectionProvider`.

```objective-c
// ViewController.h

#import <UIKit/UIKit.h>
#import "StoneSDK/StoneSDK.h"

@interface ViewController : UIViewController <STNPinPadConnectionDelegate>
@end

// Viewcontroller.m

@implementation ViewController
STNPinpadConnectionProvider* connection;

- (void)viewDidLoad
{
    [super viewDidLoad];

    connection = [[STNPinPadConnectionProvider alloc] init];

    // Define o delegate
    connection.delegate = self;

    // Inicializa o Central Manager
    [connection startCentral];
}
@end
```

Para encontrar os pinpads BLE próximos, inicie o escaneamento. Quando um pinpad BLE for encontrado, ele retornará no método `didFindPinpad` do delegate. Com este objeto STNPinpad é possível conectar-se a ele. Ao conectar-se ao pinpad desejado, é indicado parar o escaneamento.

```objective-c
- (void)startScanning
{
  // Inicia o escaneamento de pinpads próximos
  [connection startScan];
}

-(void)pinpadConnectionProvider:(STNPinPadConnectionProvider*)provider didStartScanning:(BOOL)success error:(NSError*)error
{
  // Verifica se o escaneamento foi inicializado com sucesso
}

-(void)pinpadConnectionProvider:(STNPinPadConnectionProvider*)provider didFindPinpad:(STNPinpad*)pinpad
{
  if([pinpad.name isEqualToString: @"ID_DO_PINPAD"])
  {
    [connection connectToPinpad:pinpad];
    [connection stopScan];
  }
}

-(void)pinpadConnectionProvider:(STNPinPadConnectionProvider*)provider didConnectPinpad:(STNPinpad*)pinpad error:(NSError* _ Nullable)error
{
  // Confirma a conexão com um STNPinpad
}

-(void)pinpadConnectionProvider:(STNPinPadConnectionProvider*)provider didChangeCentralState:(CBManagerState)state
{
  // Atualiza de acordo com mudanças no stado do Central manager
  // 0 - Unknown
  // 1 - Resetting
  // 2 - Unsupported
  // 3 - Unauthorized
  // 4 - Powered off
  // 5 - Powered on
}
```

Para desconectar, chame `disconnectPinpad`

```objective-c
- (void)disconnectAllPinpad
{
  // Lista todos os pinpads conectados, BLE ou clássico
    NSArray<STNPinpad*>* connectedPinpads = [connection listConnectedPinpads];

    for (STNPinpad* pinpad in connectedPinpads) {
      // Desconecta o pinpad
        [connection disconnectPinpad:pinpad];
    }
}

-(void)pinpadConnectionProvider:(STNPinPadConnectionProvider*)provider didDisconnectPinpad:(STNPinpad*)pinpad
{
  // Confirma a desconexão com um STNPinpad
}
```

É possível conectar mais de um BLE, e selecionar o que vai ser usado.

```objective-c
  [connection selectPinpad:pinpad];
```

O provider também fornece algumas consultas:

```objective-c

  // Indica se o escaneamento de BLE está em execução
  BOOL isScanning = [connection isScanning];

  // Informa o estado do Central Manager
  STNCentralState centralState = [connection centralState];

  // Verifica se o pinpad do tipo STNPinpad está conectado
  BOOL isConnected = [connection isPinpadConnected:pinpad];

  // Retorna o pinpad selecionado
  STNPinpad* selected = [connection selectedPinpad];

  // Retorna uma lista de pinpads com identificadores conhecidos
  NSArray<STNPinpad*> *pinpadList = [connection listPinpadsWithIdentifiers:@[@"ID1", @"ID2"]];
```

### Ativação do Stone Code

O provider `STNStoneCodeActivationProvider` é responsavel por ativar e desativar o lojista e possui os métodos `activateStoneCode:withblock:`, `deactivateMerchant:` e `deactivateMerchantWithStoneCode`.

> O Stone Code deve ser ativado antes de realizar qualquer operação na Stone.

Para ativar o lojista no aplicativo deve ser usado o método `activateStoneCode:withblock:`, que recebe uma string com o Stone Code do lojista como parâmetro.

```objective-c
NSString *stoneCode = @"999999999"; // Stone Code do lojista

[STNStoneCodeActivationProvider activateStoneCode:stoneCode withBlock:^(BOOL succeeded, NSError* error)
{
    if (succeeded) // verifica se a requisição ocorreu com sucesso
    {
        // em caso de sucesso,
        // executa alguma coisa
    } else
    {
        // em caso de erro,
        // faz alguma tratativa
        NSLog(@"%@", error.description);
    }
}];
```

Uma das opções para desativar o lojista no aplicativo é o método `deactivateMerchant:`, que recebe o lojista a ser desativado (um objeto do tipo `STNMerchantModel`) como parâmetro.

> Esse método excluirá o lojista do applicativo junto de todas as transações realizadas pelo mesmo.

```objective-c
STNMerchantModel *merchant = [STNMerchantListProvider listMerchants][0]; // Primeiro lojista da lista

[STNStoneCodeActivationProvider deactivateMerchant:merchant];
```

Outra opção para desativar o lojista no aplicativo é o método `deactivateMerchantWithStoneCode:`, que recebe o Stone Code por parâmetro.

```objective-c
NSString *stoneCode = @"999999999"; // Stone Code do lojista

[STNStoneCodeActivationProvider deactivateMerchantWithStoneCode:stoneCode];
```

#### Possíveis códigos de erro

[101, 202, 209](#códigos-de-erro)

### Download das tabelas AID e CAPK

O provider `STNTableDownloaderProvider` possui o método `downloadTables` que faz o download das tabelas AID e CAPK para o dispositivo iOS.

> As tabelas AID e CAPK são necessarias para fazer transações EMV.

```objective-c
[STNTableDownloaderProvider downLoadTables:^(BOOL succeeded, NSError* error)
{
    if (succeeded) // verifica se a requisição ocorreu com sucesso
    {
			// em caso de sucesso,
			// executa alguma coisa
    } else
    {
				// em caso de erro,
        // trata o erro
        NSLog(@"%@", error.description);
    }
}];
```

#### Possíveis códigos de erro

[101, 201, 601](#códigos-de-erro)

### Carregamento das tabelas AID e CAPK para o pinpad

O provider `STNTableLoaderProvider` possui o método `loadTables` que faz o update das tabelas baixadas no dispositivo iOS para o pinpad.

```objective-c
[STNTableLoaderProvider loadTables:^(BOOL succeeded, NSError* error)
{
    if (succeeded) // verifica se a requisição ocorreu com sucesso
    {
			// em caso de sucesso,
			// executa alguma coisa
    } else
    {
			// em caso de erro,
			// trata o erro
        NSLog(@"%@", error.description);
    }
}];
```

#### Possivel código de erro

[303, 304](#códigos-de-erro)

### Envio de transações

As transações são enviadas usando o método `sendTransaction:withBlock` do provider `STNTransactionProvider`.

O método `sendTransaction:withBlock:` deve receber um objeto `STNTransactionModel` como parâmetro. O objeto `STNTransactionModel` deve ter as seguintes propriedades definidas:

#### amount (NSNumber)

Propriedade obrigatória. É o valor da transação e deve ser passado no formato de centavos. Por exemplo: caso queira enviar uma transação no valor de `R$ 56,45`, deve ser passado um `NSNumber` contendo o valor de `5645`. Uma transação no valor de R$ 0,05 deve ser passada como `5`.

#### type (STNTransactionTypeSimplified)

Propriedade obrigatoria. Essa propriedade deve ser definida com o tipo da transação (débito ou crédito). Para isso podem ser usados os enums `STNTransactionTypeSimplifiedCredit` para crédito ou `STNTransactionTypeSimplifiedDebit` para débito.

#### instalmentAmount (STNTransactionInstalmentAmount)

Propriedade obrigatoria. Propriedade que define o número de parcelas da transação. Um dos seguintes enums devem ser usados:

- `STNTransactionInstalmentAmountOne` - para 1x (à vista)
- `STNTransactionInstalmentAmountTwo` - para 2x
- `STNTransactionInstalmentAmountThree` - para 3x
- `STNTransactionInstalmentAmountFour` - para 4x
- `STNTransactionInstalmentAmountFive` - para 5x
- `STNTransactionInstalmentAmountSix` - para 6x
- `STNTransactionInstalmentAmountSeven` - para 7x
- `STNTransactionInstalmentAmountEight` - para 8x
- `STNTransactionInstalmentAmountNine` - para 9x
- `STNTransactionInstalmentAmountTen` - para 10x
- `STNTransactionInstalmentAmountEleven` - para 11x
- `STNTransactionInstalmentAmountTwelve` - para 12x

#### instalmentType (STNInstalmentType)

Propriedade obrigatoria. Define o tipo de parcelamento que será efetuado. Um dos seguintes enums devem ser usados:

- `STNInstalmentTypeNone` - nenhum parcelamento, deve ser usado para transações à vista
- `STNInstalmentTypeMerchant` - parcelamento com o adquirente (sem juros)
- `STNInstalmentTypeIssuer` - parcelamento com o emissor (juros do emissor do cartão)

#### initiatorTransactionKey (NSString)

Propriedade **opcional**. Deve conter uma string contendo um valor único para identificação da transação. Caso não seja definido, um identificador único será gerado automaticamente.

#### shortName (NSString)

Propriedade **opcional**. Define um nome customizado que será exibido na fatura do cliente. O máximo de caracteres recomendado para que esse texto seja exibido corretamente em estratos e faturas é **11**. Caso não seja definido, será exibido o nome cadastrado para o **Stone Code** em uso.

#### merchant (NSMerchantModel)

Propriedade **opcional**. **Essa propriedade deve ser definida quando o aplicativo possuir mais de 1 Stone Code ativado**. A mesma pode ser usada para definir o lojista (Stone Code) que está passando a transação, caso tenha mais de 1. O valor default será sempre o primeiro Stone Code que foi ativado no aplicativo.

#### capture (STNTransactionCapture)

Propriedade **opcional**. Define se a transação será capturada no momento da transação. Deve receber um enum do tipo `STNTransactionCapture`. As opções são `STNTransactionCaptureYes` ou `STNTransactionCaptureNo`. O valor default será sempre o `STNTransactionCaptureYes`.

```objective-c
STNTransactionModel *transaction = [[STNTransactionModel alloc] init];

transaction.amount = [NSNumber numberWithInt:1000]; // valor correspondente a R$ 10,00
transaction.type = STNTransactionTypeSimplifiedDebit; // transação no débito
transaction.instalmentAmount = STNTransactionInstalmentAmountOne; // número de parcelas: 1
transaction.instalmentType = STNInstalmentTypeNone; // tipo de parcelamento: nenhum
transaction.shortName = @"Minha Loja"; // nome customizado na fatura
transaction.initiatorTransactionKey = @"9999999999999"; // ITK customizado

[STNTransactionProvider sendTransaction:transaction withBlock:^(BOOL succeeded, NSError* error) {
		if (succeeded) // verifica se a requisição ocorreu com sucesso
		{
				// em caso de sucesso,
				// executa alguma coisa
		} else
		{
			// em caso de erro,
			// trata o erro
			NSLog(@"%@", error.description);
		}
}];
```

#### Mensagens de notificação

Durante a execução de uma transação o pinpad pode envar mensagens de notificação. Essas mensagens são exibidas na tela do pinpad e também podem ser acessadas dentro da aplicação usando o `NSNotificationCenter`. Basta adicionar um observer antes do envio da transação disparando um método que usará a notificação recebida. A notificação vem como uma string. Abaixo um exemplo:

```objective-c
- (void)sendTransaction
{
    // adiciona o observer que executará o método 'handleNotification:'
		// o SDK provê o define 'PINPAD_MESSAGE' que possui o nome que a notificação deverá ter
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:PINPAD_MESSAGE object:nil];

    STNTransactionModel* transaction = [[STNTransactionModel alloc] init];

    transaction.amount = [NSNumber numberWithInt:1000];
    transaction.type = STNTransactionTypeSimplifiedCredit;
    transaction.instalmentAmount = STNTransactionInstalmentAmountOne;
    transaction.instalmentType = STNInstalmentTypeNone;

    [STNTransactionProvider sendTransaction:transaction withBlock:^(BOOL succeeded, NSError* error) {

			  if (succeeded) // verifica se a requisição ocorreu com sucesso
			  {
			  		// em caso de sucesso,
		  			// executa alguma coisa
		  	} else
		  	{
		  		// em caso de erro,
		  		// trata o erro
		  		NSLog(@"%@", error.description);
		  	}
        // remove o observer
        [[NSNotificationCenter defaultCenter] removeObserver:self name:PINPAD_MESSAGE object:nil];
    }];
}

- (void)handleNotification:(NSNotification*) notification
{
    // converte a notificação para string
    NSString* notificationString = [notification object];
    // imprime a string recebida
    NSLog(@"Mensagem do pinpad: %@", notificationString);
}
```

#### Possíveis códigos de erro

[105, 201, 203, 204, 205, 206, 207, 211, 214, 303, 601](#códigos-de-erro)

### Listagem de transações

O provider `STNTransactionListProvider` possui os métodos, `listTransactions:` e `listTransactionsByPan:`.

O método `listTransactions:` retorna um `NSArray` com as transações (`STNTransactionModel`) passadas no aplicativo. A ultima transação passada será sempre a primeira no array.

```objective-c
// Array de transações
NSArray *transactionsList = [STNTransactionListProvider listTransactions];

for (STNTransactionModel* transaction in transactionsList)
{
    NSLog(@"Valor da transação em centavos: %@", transaction.amount);
    NSLog(@"Status da transação: %@", transaction.statusString);
    NSLog(@"Tipo da transação: %@", transaction.typeString);
}
```

Para obter as transações filtrando por um cartão especifico, o método `listTransactionsByPan:` deve ser usado. Esse método solicitará a inserção de um cartão com chip. E retornará um array de transações dentro de um bloco.

```objective-c

[STNTransactionListProvider listTransactionsByPan:^(BOOL succeeded, NSArray* transactionsList, NSError* error)
{
    if (succeeded) // verifica se a requisição ocorreu com sucesso
    {
				for (STNTransactionModel *transaction in transactionsList)
					{
							NSLog(@"Valor da transação em centavos: %@", transaction.amount);
							NSLog(@"Status da transação: %@", transaction.statusString);
							NSLog(@"Tipo da transação: %@", transaction.typeString);
					}
    } else
    {
        // trata o erro
        NSLog(@"%@", error.description);
    }
}];
```

#### Possíveis códigos de erro

[101, 304](#códigos-de-erro)

### Listagem de lojistas

O provider `STNMerchantListProvider` possui o método `listMerchants:` que retorna um `NSArray` contendo os lojistas (`STNMerchantModel`) ativados no aplicativo.

```objective-c
// Array de transações
NSArray *merchantsList = [STNMerchantListProvider listMerchants];

for (STNMerchantModel* merchant in merchantsList)
{
    NSLog(@"Nome do lojista: %@", merchantsList.merchantName);
    NSLog(@"CPF ou CNPJ do lojista: %@", merchantsList.documentNumber);
    NSLog(@"SAK: %@", merchantsList.saleAffiliationKey);
}
```

#### Possíveis códigos de erro

[101, 304](#códigos-de-erro)

### Cancelamento de transações

O responsável pelo cancelamento das transações é o método `cancelTransaction` do provider `STNCancellationProvider`, que recebe como parâmetro o objeto de transação `STNTransactionModel`.

```objective-c
// preenche array com lista de transações
NSArray *transactionsList = [STNTransactionListProvider listTransactions];

// instacia o objeto de transação com a última transação realizada
STNTransactionModel *transaction = transactionsList[0];

// executa o cancelamento
{
		if (succeeded) // verifica se a requisição ocorreu com sucesso
		{
				// em caso de sucesso,
				// executa alguma coisa
		} else
		{
				// em caso de erro,
				// trata o erro
				NSLog(@"%@", error.description);
		}
}];
```

#### Possíveis códigos de erro

[101, 210, 601](#códigos-de-erro)

### Envio de comprovante por email

Para enviar comprovantes de transações por email basta usar o método `sendReceiptViaEmail:` do provider `STNMailProvider`.

O método `sendReceiptViaEmail:` recebe os parâmetros:

#### mailTemplate (STNMailTemplate)

O primeiro parâmetro que deve ser informado é um enum que representa o template de email, podendo ser `STNMailTemplateTransaction` para comprovantes de transação, ou `STNMailTemplateVoidTransaction` para comprovantes de cancelamento.

#### transaction (STNTransactionModel)

O parâmetro transactionInfo deve receber um objeto do provider `STNTransactionModel` que terá as informações da transação.

#### destination (NSString)

Destination deve conter uma string com o email do destinatario.

#### displayCompanyInformation (BOOL)

Esse parâmetro recebe um booleano que dirá se os dados do lojista (como endereço e CPF/CNPJ) serão exibidos no comprovante ou não.

> Alguns lojistas são pessoas físicas e querem que suas informações não sejam exibidas.

```objective-c
NSArray *transactions = [STNTransactionListProvider listTransactions];

// destinatario
NSString *destination = @"fulano@destino.com.br";

// envia email com comprovante da última transação realizada
[STNMailProvider sendReceiptViaEmail:STNMailTemplateTransaction transaction:transactions[0] toDestination:destination displayCompanyInformation:YES withBlock:^(BOOL succeeded, NSError* error)
{
		if (succeeded) // verifica se a requisição ocorreu com sucesso
		{
				// em caso de sucesso,
				// executa alguma cois
 		} else
		{
				// em caso de erro,
				// trata o erro
				NSLog(@"%@", error.description);
		}
}];
```

#### Possíveis códigos de erro

[103, 601](#códigos-de-erro)

### Validações

O provider `STNValidationProvider` possui 4 métodos de validação:

#### validateActivation

Verifica se o Stone Code já foi ativado e retorna `YES` caso positivo.

#### validatePinpadConnection

Valida se o pinpad está pareado com o dispositivo **iOS** e retorna `YES` caso positivo.

> Lembrando que para a comunicação ocorrer ainda é necessario estabelecer uma sessão.

#### validateTablesDownloaded

Checa se as tabelas AID e CAPK já foram baixadas para o dispositivo **iOS** e retorna `YES` caso positivo.

#### validateConnectionToNetWork

Verifica se a conexão com a internet está funcionando e retorna `YES` caso positivo.

```objective-c
if ([STNValidationProvider validateActivation] == YES)
{
    NSLog(@"Stone Code está ativado!");
}

if ([STNValidationProvider validatePinpadConnection] == YES)
{
    NSLog(@"O pinpad está pareado com o dispositivo iOS!");
}

if ([STNValidationProvider validateTablesDownloaded] == YES)
{
    NSLog(@"As tabelas já foram baixadas para o dispositivo iOS!");
}

if ([STNValidationProvider validateConnectionToNetWork] == YES)
{
    NSLog(@"A conexão com a internet está ativa!");
}
```

> É importante que essas validações sejam executadas e tratadas antes de realizar as operações.

### Captura de PAN

Para capturar o PAN (4 últimos dígitos do cartão) deve ser usado o método `getCardPan:` do provider `SNTCardProvider`.

```objective-c
[STNCardProvider getCardPan:^(BOOL succeeded, NSString* pan, NSError* error)
{
		if (succeeded) // verifica se a requisição ocorreu com sucesso
		{
				NSLog(@"**** **** **** %@", pan);
		} else
		{
				// em caso de erro,
				// trata o erro
				NSLog(@"%@", error.description);
		}
}];
```

#### Possíveis códigos de erro

[101, 304](#códigos-de-erro)

### Exibição no display do pinpad

Para exibir uma mensagem no display do pinpad pode ser usado o método `displayMessage:withBlock:` do provider `STNDisplayProvider` que recebe uma mensagem no formato de string. A string enviada deve conter no máximo 16 caracteres.

```objective-c
[STNDisplayProvider displayMessage:@"MINHA MENSAGEM" withBlock:^(BOOL succeeded, NSError* error)
{
		if (succeeded) // verifica se a requisição ocorreu com sucesso
		{
				// em caso de sucesso,
				// executa alguma coisa
		} else
		{
				// em caso de erro,
				// trata o erro
				NSLog(@"%@", error.description);
		}
}];
```

#### Possíveis códigos de erro

[101, 105, 304](#códigos-de-erro)

## Models

Alguns providers retornam models que podem ser usados pelo usuario do SDK.

### Transação

O model `STNTransactionModel` disponibiliza, em suas propriedades, informações de uma transação.

#### Lista de propriedades

- amount (**NSNumber**) - valor da transação no formato de centavos (ex: 10,00 vai ser 1000. Basta multiplicar por 0.01 para obter o valor real.)
- instalmentAmount (**STNTransactionInstalmentAmount**) - número de parcelas da transação
- balance (**NSNumber**) - saldo do voucher (ex.: Ticket, Sodexo)
- instalmentType (**STNInstalmentType**) - tipo de parcelamento da transação
- aid (**NSString**) - Código AID da transação
- arqc (**NSString**) - código ARQC da transação
- type (**STNTransactionTypeSimplified**) - Débito ou crédito
- typeString (**NSString**) - string que representa a propriedade `type`
- status (**STNTransactionStatus**) - aprovada, cancelada, negada...
- statusString (**NSString**) - string que representa a propriedade `status`
- date (**NSDate**) - Data da transação
- dateString (**NSString**) - string que representa a propriedade `date`
- receiptTransactionKey (**NSString**) - ID da transação
- reference (**NSString**) - referencia da transação
- pan (**NSString**) - 4 últimos número do cartão
- cardBrand (**NSString**) - bandeira do cartão
- cardHolderName (**NSString**) - nome do portador do cartão
- authorizationCode (**NSString**) - Stone ID
- initiatorTransactionKey (**NSString**) - identificação da transação
- shortName (**NSString**) - nome customizado exibido na fatura (se não for definido será `nil`)
- merchant (**STNMerchantModel**) - lojista que passou a transação
- pinpad (**STNPinpadModel**) - pinpad que passou a transação

### Lojista

O model `STNMerchantModel` disponibiliza, em suas propriedades, informações do lojista/usuario do aplicativo.

#### Lista de propriedades

- saleAffiliationKey (**NSString**) - Afiliation key
- documentNumber (**NSString**) - CPF/CNPJ
- merchantName (**NSString**) - Nome do lojista
- stonecode (**NSString**) - Stone Code
- address (**STNAddressModel**) - Endereço do lojista
- transactions (**NSOrderedSet<STNTransactionModel>**) - transações do lojista

### Pinpad

O model `STNPinpadModel` disponibiliza, em suas propriedades, informações do pinpad.

#### Lista de propriedades

- name (**NSString**) - nome
- model (**NSString**) - modelo
- serialNumber (**NSString**) - número de serie
- transaction (**STNTransactionModel**) - transação passada com o pinpad

### Endereço

O model `STNAddressModel` disponibiliza, em suas propriedades, informações de endereço.

#### Lista de propriedades

- city (**NSString**) - cidade
- district (**NSString**) - estado
- neighborhood (**NSString**) - bairro
- street (**NSString**) - rua
- doorNumber (**NSString**) - número
- complement (**NSString**) - complemento
- zipCode (**NSString**) - CEP
- merchant (**STNMerchantModel**) - lojista que possui esse endereço


## Outros

### Pinpad

O `STNPinpad` é um objeto representativo do pinpad.

#### Lista de propriedades

- name (**NSString**) - nome do dispositivo
- identifier (**NSString**) - o UUID de dispositivos BLE, ou o connection ID de dispositivos Bluetooth
- alias (**NSString**): nome customizado do dispositivo
- device (**id**): guarda o objeto CBPeripheral (BLE) ou EAAccessory (Bluetooth)

### Códigos de erro

- 101 - erro genérico
- 103 - falha no envio de email
- 105 - número de caracteres acima do permitido
- 106 - número de caracteres acima do permitido para a propriedade `shortName`
- 110 - erro no comando FNC
- 201 - falta ativar o Stone Code
- 202 - Stone Code informado já foi ativado
- 203 - valor informado para transação é inválido
- 204 - transação cancelada durante operação
- 205 - transação inválida
- 206 - falha na transação
- 207 - tempo da transação expirado
- 209 - Stone Code desconhecido
- 210 - Transação já foi cancelada
- 211 - transação negada
- 214 - operação cancelada pelo usuario
- 303 - conexão com o pinpad não encontrada
- 304 - tabelas AID e CAPK não encontradas
- 305 - erro ao carregar tabelas para o pinpad
- 306 - erro no request
- 601 - erro na conexão com a internet
