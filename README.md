# Relatório de Incidente — Cenário E: Réplica Isolada e Queda do Primário

## 1. Identificação
* **Participante(s):** Pedro Ivo da Cunha Martins de Alencar
* **Cenário Sorteado:** Cenário E (Réplica isolada e depois queda do primário)
* **Ambiente Utilizado:** PostgreSQL 18 (Instalação nativa no Windows / Caminho B)
* **Portas das instâncias:** Primário (`5442`), Réplica (`5443`)

---

## 2. Resumo do Incidente
Neste cenário, simulou-se a interrupção da rede entre o servidor primário e a réplica (isolamento). Com a réplica desconectada, o primário continuou recebendo transações de escrita e inserção de eventos. Minutos depois, o primário sofreu uma falha abrupta e caiu. Como a réplica estava desatualizada devido ao isolamento, ao ser promovida a novo primário, ocorreu perda de dados recentes (**RPO**). Posteriormente, o ambiente foi recriado aplicando a **replicação síncrona** (`synchronous_standby_names = '*'`) para zerar a perda de dados, observando-se o comportamento de travamento do primário quando a réplica fica offline.

---

## 3. Linha do Tempo

| Horário | Ação Executada | Resultado / Estado do Sistema |
| :--- | :--- | :--- |
| `[HH:MM:SS]` | Desligamento da réplica (`pg_ctl -D replica -m fast stop`) | Réplica isolada (simulando corte de rede). |
| `[HH:MM:SS]` | Inserção de 50 eventos no primário (`INSERT INTO eventos ...`) | Dados gravados apenas no primário. Anotação do ID máximo ($M_1$). |
| `[HH:MM:SS]` | Queda abrupta do primário (`pg_ctl -D primary -m fast stop`) | Anotação do horário $T_0$. Primário e réplica inativos. |
| `[HH:MM:SS]` | Inicialização e promoção da réplica (`pg_promote()`) | Réplica promovida a novo primário em $T_1$. |
| `[HH:MM:SS]` | Consulta do ID máximo no novo primário ($M_2$) | Constatação da perda de dados equivalentes aos eventos inseridos no isolamento. |

---

## 4. Medições (RTO e RPO)

* **RPO (Recovery Point Objective - Perda de Dados):**
  * Fórmula: $\text{RPO} = M_1 - M_2$
  * Cálculo: $M_1 \text{ (último ID no primário antes da queda)} - M_2 \text{ (último ID recuperado na réplica promovida)}$.
  * **Resultado:** Perda de `50` registros (as linhas inseridas enquanto a réplica estava isolada).

* **RTO (Recovery Time Objective - Tempo de Recuperação):**
  * Fórmula: $\text{RTO} = T_1 - T_0$
  * Cálculo: Tempo decorrido desde o momento da queda do primário ($T_0$) até a subida e comando de promoção da réplica ($T_1$).
  * **Resultado:** Aproximadamente `[X minutos / segundos]`.

---

## 5. Causa Raiz
A perda de dados ocorreu porque a instância operava em **modo assíncrono** por padrão. No modo assíncrono, o primário confirma o commit da transação localmente antes de garantir que o log de transações (WAL) foi gravado e aplicado na réplica. Quando a conexão foi cortada (isolamento) e o primário falhou, os dados que não viajaram pela rede deixaram de existir no ponto de recuperação da réplica.

---

## 6. Correção Aplicada e Prova de Funcionamento
Para garantir **RPO = 0**, o ambiente foi reinicializado e configurou-se o parâmetro de replicação síncrona no primário:
```sql
ALTER SYSTEM SET synchronous_standby_names = '*';
SELECT pg_reload_conf();

Prova de funcionamento: Com a réplica desligada, tentou-se realizar uma inserção no primário:

SQL
CREATE TABLE teste_sincrono (id serial);
INSERT INTO teste_sincrono DEFAULT VALUES;
Resultado observado: O comando travou (congelou) aguardando a confirmação da réplica, comprovando que o primário não efetiva commits sem a garantia síncrona. Em seguida, a configuração foi revertida (ALTER SYSTEM RESET synchronous_standby_names;) para restabelecer a flexibilidade do laboratório.

7. Comparação com Serviço Gerenciado
Em provedores de nuvem (como AWS RDS Multi-AZ):

Multi-AZ: A replicação síncrona e o failover são gerenciados de forma transparente. Se o primário falha, a nuvem promove a instância standby automaticamente (atualizando o DNS/IP virtual), mitigando o RTO para poucos segundos e garantindo RPO próximo de zero sem intervenção manual.

PITR (Point-in-Time Recovery): Permite restaurar o banco de dados para qualquer segundo exato usando backups automáticos combinados com os arquivos WAL contínuos, protegendo contra exclusões acidentais ou corrupções lógicas.

8. Prevenção e Lições
Monitoramento de Lag: Implementar alertas automatizados para identificar atrasos ou desconexões em slots de replicação.

Avaliação de Trade-offs: Ponderar a alta disponibilidade versus o risco de paralisação total da aplicação antes de adotar replicação síncrona estrita em ambientes de alta volatilidade de rede.

Simulações Periódicas: Executar testes regulares de failover para certificar os tempos reais de RTO e RPO da equipe.

9. Respostas às Perguntas do Cenário
Com síncrona, qual o RPO e o que se paga (latência, disponibilidade)?

RPO: Zero (0), pois nenhuma transação é confirmada sem cópia na réplica.

O que se paga: Paga-se em latência (maior tempo de resposta devido à espera pela rede) e em disponibilidade (se a réplica cair, o primário para de aceitar novas gravações).

Em que sistema usaria síncrona e em qual assíncrona?

Síncrona: Sistemas transacionais estritos, como instituições financeiras, meios de pagamento e controle de estoque de alta precisão.

Assíncrona: Sistemas de logs, redes sociais para postagens e feeds, ou análises onde a prioridade máxima é manter o sistema sempre rápido e online.

Que alarme avisaria que a réplica desconectou?

Um alarme baseado na métrica da view pg_stat_replication verificando se o campo state é diferente de streaming, ou monitorando o atraso de bytes através de ferramentas de observabilidade como Prometheus, Grafana ou Zabbix.