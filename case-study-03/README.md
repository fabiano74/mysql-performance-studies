## Estudo de Caso #3: Otimização do Fluxo de aprovação com vários status e mensagens ao usuário

### 1. O Desafio (Problema Original)

O ERP teve a inclusão de novos status no módulo de Pagamento, o que ocasionou um problema no fluxo da trigger para exclusão de contas.


#### 1.1  Problema Arquitetural
* **Problema Arquitetural:** A inserção de novos *status* de fluxo (como `unprocessed` ou *status* de controle) que não representam movimentação financeira quebrou a premissa da *trigger*. O código bloqueava incorretamente *status* que deveriam ser excluíveis (se fossem rascunhos) ou dava a mensagem errada.
* **Necessidade:** Mudar a lógica de checagem da exclusão para uma abordagem de **Categorização do Status**. Isso permite que o código acomode futuros *status* sem quebrar a regra de integridade, ao mesmo tempo que melhora o feedback ao usuário (UX).

### 2. Solução Implementada (Lógica Condicional IF/ELSEIF)
A solução foi utilizar blocos condicionais aninhados (`IF / ELSEIF`) para categorizar os status, mantendo a integridade máxima (bloqueando todos os status que não são rascunho) com mensagens específicas.

```sql
CREATE DEFINER=`admin`@`%` TRIGGER `DeleteConta` BEFORE DELETE ON `CONTAS` FOR EACH ROW 
BEGIN

  -- Bloqueia exclusão se o status tiver movimentação financeira'
   IF OLD.Status IN ('pago', 'parcial') THEN
       SIGNAL SQLSTATE '45000'
       SET MESSAGE_TEXT = 'Exclusão não permitida. O registro possui movimentação financeira.';
 
  -- BLOQUEIO POR FALHA EXTERNA (Erro Bancário)
   ELSEIF OLD.Status IN ('nao_processado') THEN
       SIGNAL SQLSTATE '45000'
       SET MESSAGE_TEXT = 'Exclusão não permitida. O registro foi devolvido pelo banco e requer tratamento de auditoria.';
 
  -- Bloqueia a exclusão na movimentação de fluxo de aprovação de contas
  ELSEIF OLD.Status IN ('aprovado', 'aprovacao', 'processando', 'recusado') THEN
  		SIGNAL SQLSTATE '45000'
  		SET MESSAGE_TEXT = 'O registro está dentro do fluxo de aprovação e não pode ser excluído.';
 
  END IF;

END
 ```

### 3.  Resultado e Impacto
Integridade Garantida: Todos os status que representam valor contábil ou registro de auditoria são protegidos.

UX Aprimorada: O usuário recebe uma mensagem clara que o orienta (por exemplo, indicando que deve haver um estorno para faturas paid). O feedback do erro no ERP se torna mais informativo e profissional..



## 1. O Desafio (Impacto da Evolução do Fluxo de Status)

A *trigger* original para bloqueio de exclusão em `CONTAS` usava a lógica para status focados somente no fluxo financeiro: `IF OLD.Status <> 'pendente' THEN BLOCK`.

* **Problema Arquitetural:** A inserção de novos *status* de fluxo (como `nao_processado` ou *status* de controle) que não representam movimentação financeira quebrou a premissa da *trigger*. O código bloqueava incorretamente *status* que deveriam ser excluíveis (se fossem rascunhos) ou dava a mensagem errada.
* **Necessidade:** Mudar a lógica de checagem da exclusão para uma abordagem de **Categorização do Status**. Isso permite que o código acomode futuros *status* sem quebrar a regra de integridade, ao mesmo tempo que melhora o feedback ao usuário (UX).

