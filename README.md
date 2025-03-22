# Novidades RAP - Copy Instance
E que tal poder copiar uma linha com dados já existentes, alterar apenas o que é necessário em modo editar e salvar? Ou mesmo, copiar uma linha e em tempo de execução alterar algum campo e já mostrar o novo cadastro em modo preview? Podemos sim utilizar este recurso no RAP + Fiori elements. Quem nunca recebeu solicitação de usuário para ter um botão com este recurso!? Agora temos como facilitar a vida e ganhar uns créditos.

Neste forum falo mais especificamente SAP S/4HANA, on-premise edition or SAP S/4HANA Cloud, private edition (https://help.sap.com/docs/identity-provisioning/identity-provisioning/proxy-sap-s-4hana-on-premise) e como utlizar a feature, no final passo o código exemplo de todos os objetos criados para ter o cenário do Copy instance, dentre eles: Database table, CDS, BDEF...

> [!NOTE]
> Verificar versão da feature BDEF no link: https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENRAP_FEATURE_TABLE.html.
![image](https://github.com/user-attachments/assets/108242d1-6313-47e0-ad87-509b88fd0a18)
![image](https://github.com/user-attachments/assets/4d0015cc-0087-42cf-bf93-cab7670e4fec)

# Factory action no Behavior Definition (BDEF) e Implementar na Classe
Defina a feature no BDEF, clique CTRL + 1 para abrir o Quick assist, e definir na classe automáticamente o método da action
![image](https://github.com/user-attachments/assets/3bca7727-bdea-4d18-a585-01cb8a4df5f1)

```
    // instance factory action for copying SO instances
    factory action copyInstance [1];
```
# composition-child
Composition child exemple
```
    // instance factory action for copying SO instances
    factory action copyInstance [1];
```
