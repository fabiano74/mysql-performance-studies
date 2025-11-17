## Estudo de Caso #1: Otimização de Regra de Exclusão (DDL/Trigger)

### 1. O Desafio (Problema Original)

A lógica de exclusão de contas dependia da existência de uma **Chave Estrangeira (FK_REGISTRO_EXTERNO)** para validar se a conta tinha movimentação. Este método resultava em lógica ambígua e potencial para consultas lentas dentro do gatilho.

#### 1.1. Comando SQL (Antigo - Abstrato)

```sql
-- Lógica Antiga: Dependente de consulta externa (custo de performance)
CREATE TRIGGER TGR_DELETE_CONTA_OLD
BEFORE DELETE ON TB_CADASTRO_CONTA
FOR EACH ROW
BEGIN
    IF EXISTS (SELECT 1 FROM TB_MOVIMENTACAO WHERE ID_MOV = OLD.FK_REGISTRO_EXTERNO) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Conta já movimentada.';
    END IF;
END; ```


### 2. Solução Implementada (Melhoria Arquitetural)
A regra foi refatorada para validar a exclusão com base no Status de Processamento interno do próprio registro, tornando a validação imediata e robusta.
#### 2.1. Comando SQL (Novo - Abstrato)

``` sql
-- Lógica Nova: Validação Interna e Direta (ganho de performance e clareza)
CREATE TRIGGER TGR_DELETE_CONTA_NEW
BEFORE DELETE ON TB_CADASTRO_CONTA
FOR EACH ROW
BEGIN
    IF OLD.STATUS <> 'PENDING’' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Existe movimentação nesta conta e não pode ser excluída.';
    END IF;
END; ```

### 3. Benefício de Engenharia e Performance
Integridade Aprimorada: A regra de negócio se tornou auto-contida e mais robusta, baseada no estado de vida do registro (Status) e não em uma dependência externa.
Performance (Tuning): Eliminação da necessidade de uma consulta externa (SELECT EXISTS) dentro do TRIGGER, garantindo execução instantânea da validação e menor latência nas operações de exclusão.

