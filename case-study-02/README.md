## Estudo de Caso #2: Otimização do Fluxo de Trabalho (workflow) com Processamento em Lote

### 1. O Desafio (Problema Original)

O processamento de aprovações e transições de status em sistemas de Contas a Pagar exigia múltiplas requisições ao banco de dados (round trips) para aprovar diversos lançamentos de uma só vez. Além disso, a lógica de transição e validação do status de origem ficava dispersa na camada da aplicação (ERP), o que gerava latência em aprovações em lote e comprometia a integridade dos dados no banco.

#### 1.1. Comando SQL (Antigo - Abstrato)

```sql
-- Lógica Antiga: Múltiplas Chamadas (Executadas dentro de um loop no ERP)
-- Alto overhead de rede e transações para grandes volumes.

-- Primeiro, validação do status:
-- SELECT Status FROM FINANCIAL.BILLS WHERE ID = :id_atual; 

-- Depois, o update:
-- IF :status_retornado = 'for_approval' THEN
--     UPDATE FINANCIAL.BILLS SET Status = 'approved_launch' WHERE ID = :id_atual;
-- END IF;
 ```


### 2. Solução Implementada (Melhoria Arquitetural)
A solução foi centralizar todo o fluxo de transição de status (forward, approve, refuse, processing) em uma única Stored Procedure (ApprovalFlow) e otimizar a consulta para aceitar múltiplos IDs em uma única chamada.

```sql
-- Lógica Nova: SP robusta, centralizada e otimizada para LOTE.

CREATE DEFINER=`admin`@`%` PROCEDURE `FINANCIAL`.`ApprovalFlow`(P_IDENTIFIER TEXT, P_ACTION  VARCHAR(50))
BEGIN
	
    -- verifica se o lançamento foi encontrado
    IF P_IDENTIFIER IS NOT NULL THEN
		
    CASE P_ACTION
        WHEN 'forward' THEN

            -- Regra: Só pode encaminhar se estiver PENDENTE.            
                UPDATE FINANCIAL.BILLS SET Status = 'for_approval' WHERE FIND_IN_SET(IDFB, P_IDENTIFIER)
                AND Status = 'pending';
            
        WHEN 'approve' THEN

            -- Regra: Só pode aprovar se estiver PARA PAGAMENTO.            
                UPDATE FINANCIAL.BILLS SET Status = 'approved_launch' WHERE FIND_IN_SET(IDFB, P_IDENTIFIER)
                AND Status = 'for_approval';
            
        WHEN 'refuse' THEN

            -- Regra: Só pode recusar se estiver PARA PAGAMENTO.           
                UPDATE FINANCIAL.BILLS SET Status = 'refused_launch'  WHERE FIND_IN_SET(IDFB, P_IDENTIFIER)
                AND Status = 'for_approval';
            
        WHEN 'processing' THEN

            -- Regra: Só pode preparar para o banco se estiver LCTO APROVADO.            
                UPDATE FINANCIAL.BILLS SET Status = 'in_processing' WHERE FIND_IN_SET(IDFB, P_IDENTIFIER)
                AND Status = 'approved_launch';
          
    END CASE;
    
	END IF;
END
 ```

### 3. Benefício de Engenharia e Performance
Performance (Tuning): A utilização da função FIND_IN_SET permite processar centenas de IDs com uma única chamada SQL ao invés de N chamadas. Isso elimina o overhead de latência de rede e reduz significativamente o throughput (tempo total de processamento) para operações em lote.

Integridade e Engenharia: A lógica de negócio para a transição de status (ex: "só pode aprovar se estiver for_approval") foi migrada para o banco de dados. Isso garante que as regras sejam aplicadas de forma transacional e atômica, prevenindo erros de integridade causados por lógicas desatualizadas na aplicação ou chamadas diretas não validadas.

Centralização: Simplifica a manutenção e futuras alterações no fluxo, que agora são realizadas em um único ponto (a Stored Procedure), e não em múltiplos endpoints da aplicação.

