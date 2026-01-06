---
title: "Configurando o seu ambiente AWS"
tags: ["AWS"]
date: 2026-01-05
draft: false
---

Uma vez que você criou sua conta na AWS, é considerado uma boa prática não usar a sua conta principal para tarefas do dia a dia. Para evitar o uso da conta principal é recomendado criar um usuário alternativo com acesso aos recursos necessários

No nosso caso podemos listar recursos específicos, mas como o objetivo é estudar os recursos da AWS, eu gerei um novo usuário com recursos administrativos

Uma vez logado na conta principal, busque o Identity Access Management (IAM) para criar o novo usuário
![IAM](img/user1.png)

## Crie um novo usuário
* Dê um nome ao seu usuário e marque a opção <i><strong>Provide user access to the AWS Management Console - optional</strong></i> para que o usuário possa se logar no console da AWS
* No meu caso específico, desejo criar um usuário com o qual eu consiga fazer deploy de recursos da minha máquina pessoal com IaC (Infrastructure as Code), por isso marquei a policy <i><strong>AdministratorAccess</strong></i>
* Crie o usuário selecionando o botão <i><strong>Create User</strong></i>
* Após a criação do usuário é possível recuperar a senha temporária

{{<gallery>}}
  <img src="img/user2.png" class="grid-w25">
  <img src="img/user3.png" class="grid-w25">
  <img src="img/user4.png" class="grid-w25">
  <img src="img/user5.png" class="grid-w25">
{{</gallery>}}

## Obtenha a Chave de Acesso
Uma vez que seu usuário foi criado, selecione o usuário e o campo <b><i>Access Key</i></b> na tela
{{<figure src="img/access-key1.png">}}

Selecione a opção <b><i>Command Line Interface (CLI)</i></b> e baixe o arquivo csv ou copie a chave de acesso. Esta informação será necessária para a configuração das credenciais da AWS no seu ambiente de desenvolvimento
{{<gallery>}}
<img src="img/access-key2.png" class="grid-w25">
<img src="img/access-key3.png" class="grid-w33">
<img src="img/access-key4.png" class="grid-w33">
{{</gallery>}}

<hr />

## Primeiro Login

> [!info]+ Primeiro Login
> Uma vez que o usuário foi criado, faça o primeiro acesso e gere a nova senha

{{<gallery>}}
  <img src="img/login1.png" class="grid-w33">
  <img src="img/login2.png" class="grid-w33">
{{</gallery>}}

<hr />

## Configure o ambiente local

Uma vez que você tenha criado o novo usuário e obtido a chave de acesso, podemos então criar os arquivos necessários para o ambiente local

{{<tabs>}}
  {{<tab label="Linux / macOS">}}
  ```shell
  mkdir ~/.aws
  cd ~/.aws
  
  echo [default] > credentials
  echo aws_access_key_id=<access key> >> credentials
  echo aws_secret_access_key=<access secret> >> credentials

  echo [default] > config
  echo region=us-east-1 >> config
  echo output=json >> config
  ```
  {{</tab>}}
  {{<tab label="Windows">}}
  ```powershell
  mkdir C:\Users\USERNAME\.aws
  cd C:\Users\USERNAME\.aws
  
  echo [default] > credentials
  echo aws_access_key_id=<access key> >> credentials
  echo aws_secret_access_key=<access secret> >> credentials

  echo [default] > config
  echo region=us-east-1 >> config
  echo output=json >> config
  ```  
  {{</tab>}}
{{</tabs>}}