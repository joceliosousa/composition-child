# Novidades RAP - Factory action: Copy Instance
E que tal poder copiar uma linha com dados já existentes, alterar apenas o que é necessário em modo editar e salvar? Ou mesmo, copiar uma linha e em tempo de execução alterar algum campo e já mostrar o novo cadastro em modo preview? Já Podemos utilizar este recurso combinando RAP + Fiori elements. 

Quem nunca recebeu solicitação de usuário para ter um botão com este recurso!? Agora temos como facilitar a vida e ganhar uns créditos, ou não!?

Neste fórum falo mais especificamente SAP S/4HANA, on-premise edition or SAP S/4HANA Cloud, private edition (https://help.sap.com/docs/identity-provisioning/identity-provisioning/proxy-sap-s-4hana-on-premise) e como utlizar a feature. No final passo o código exemplo de todos os objetos criados para ter o cenário do Copy instance, dentre eles: Database table, CDS, BDEF...

> [!NOTE]
> Verificar versão da feature BDEF no link: https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENRAP_FEATURE_TABLE.html.
![image](https://github.com/user-attachments/assets/108242d1-6313-47e0-ad87-509b88fd0a18)
![image](https://github.com/user-attachments/assets/4d0015cc-0087-42cf-bf93-cab7670e4fec)

## Factory action no Behavior Definition (BDEF) e Implementar na Classe
Defina a feature no BDEF, clique CTRL + 1 para abrir o Quick assist e definir na classe o método relativo a entity-action na classe, automáticamente. A Classe relativa ao BDEF mostrará o novo método privado e sua implementação, que no momento está vazia.
![image](https://github.com/user-attachments/assets/b4f70d63-4c95-4098-b123-7aa3f643252f)

![image](https://github.com/user-attachments/assets/d697f799-3865-425b-a118-a79eb8c3dcd8)

Abaixo o código relativo a definição da factory action no BDEF e implementação do método CopyInstance.

```
    // instance factory action for copying SO instances
    factory action copyInstance [1];
```

```
  METHOD copyinstance.

    DATA: header    TYPE TABLE FOR CREATE yi_header_info,
          items_cba TYPE TABLE FOR CREATE yi_header_info\\header\_item.

    " Reading selected data from frontend
    READ ENTITIES OF yi_header_info IN LOCAL MODE
    ENTITY header
    ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(roots)
    FAILED failed.

    READ ENTITIES OF yi_header_info IN LOCAL MODE
    ENTITY header BY \_item
    ALL FIELDS WITH CORRESPONDING #( roots )
    RESULT DATA(items)
    FAILED failed.

    LOOP AT roots ASSIGNING FIELD-SYMBOL(<lfs_root>).

      APPEND VALUE #( %cid = keys[ KEY entity %key = <lfs_root>-%key ]-%cid
                      %is_draft = keys[ KEY entity %key = <lfs_root>-%key ]-%param-%is_draft
                      %control-hdrkey = if_abap_behv=>mk-off
                      %data = CORRESPONDING #( <lfs_root> EXCEPT hdrkey )
       )  TO header ASSIGNING FIELD-SYMBOL(<lfs_header>).
       <lfs_header>-invreno += 1.

      "Fill %cid_ref of header as instance identifier for cba item
      APPEND VALUE #( %cid_ref = keys[ KEY entity %key = <lfs_root>-%key ]-%cid
                      %is_draft = keys[ KEY entity %key = <lfs_root>-%key ]-%param-%is_draft
                    )
        TO items_cba ASSIGNING FIELD-SYMBOL(<items_cba>).

      LOOP AT items ASSIGNING FIELD-SYMBOL(<item>) USING KEY entity WHERE hdrkey EQ <lfs_root>-hdrkey.
        "Fill item container for creating item with cba
        APPEND VALUE #( %cid      = keys[ KEY entity %key = <lfs_root>-%key ]-%cid && <item>-itmkey
                        %is_draft = keys[ KEY entity %key = <lfs_root>-%key ]-%param-%is_draft
                        %data     = CORRESPONDING #( items[ KEY entity %key = <item>-%key ] EXCEPT hdrkey itmkey )
                      )
          TO <items_cba>-%target ASSIGNING FIELD-SYMBOL(<new_item>).
      ENDLOOP.

    ENDLOOP.

    "Create BO Instance by COPY
    MODIFY ENTITIES OF yi_header_info IN LOCAL MODE
    ENTITY header
    CREATE FIELDS ( invreno
                    shiploc
                    salesarea
                    partno
                    shipstatus )
    WITH header
    CREATE BY \_item FIELDS ( itemno
                              itmstatus
                              salesoffice
                              uom
                              delvqty
                              aprqty )
      WITH items_cba
    MAPPED mapped.

  ENDMETHOD.

```
No exemplo mostrado acima, ele vai copiar não só as linha do cabeçalho mas também os itens da child **Items**.
Neste cenário, os dados copiados vão aparecer em tempo de em modo edição - DRAFT, logo após a copia. Se quizer fazer a cópia da linha selecionada mostrando no modo preview, apenas retire a referências: %is_draft = keys[ KEY entity %key = <lfs_root>-%key ]-%param-%is_draft, onde aparecer nesta implementação.

Neste cenário, que passo no final todos os objetos, foi utilizado o recurso field ( numbering : managed ) tanto na Root como na cada child Items. Também incremento o campo invreno com + 1 para não repetir o mesmo número, apesar de não ser chave. Este campo seria o nosso número da Invoice. A child Billing utiliza um recurso na hora de salvar (with additional save), que explico em outro momento. Existe um botão **Create Billing from Item** na child Items que tem como objetivo criar itens de Billing relativos a todos os Itens, que explicarei em outro momento.

**O resultado é este**:
![image](https://github.com/user-attachments/assets/d9c303aa-e81c-4fe1-b6fa-d96ed08f2143)


![image](https://github.com/user-attachments/assets/2f1229e8-7abd-4309-9cd0-46fde23c9731)



## composition-child
Composition child exemple
```
    // instance factory action for copying SO instances
    factory action copyInstance [1];
```
