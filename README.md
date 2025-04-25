# Novidades RAP - Factory action: Copy Instance
E que tal poder copiar uma linha com dados já existentes, alterar apenas o que é necessário em modo editar e salvar? Ou mesmo, copiar uma linha e em tempo de execução alterar algum campo e já mostrar o novo cadastro em modo preview? Já Podemos utilizar este recurso combinando RAP + Fiori elements. 

Quem nunca recebeu solicitação de usuário para ter um botão com este recurso!? Agora temos como facilitar a vida e ganhar uns créditos, ou não!?

Neste fórum falo mais especificamente SAP S/4HANA, on-premise edition or SAP S/4HANA Cloud, private edition (https://help.sap.com/docs/identity-provisioning/identity-provisioning/proxy-sap-s-4hana-on-premise) e como utlizar a feature. No final passo o código exemplo de todos os objetos criados para ter o cenário do Copy instance, dentre eles: Database table, CDS, BDEF...

> !NOTE
> Verificar versão da feature BDEF no link: https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/ABENRAP_FEATURE_TABLE.html.
![image](https://github.com/user-attachments/assets/108242d1-6313-47e0-ad87-509b88fd0a18)
![image](https://github.com/user-attachments/assets/4d0015cc-0087-42cf-bf93-cab7670e4fec)

## Factory action no Behavior Definition (BDEF) e Implementar na Classe
Defina a feature no BDEF, clique CTRL + 1 para abrir o Quick assist e definir na classe o método relativo a entity-action na classe, automáticamente. A Classe relativa ao BDEF mostrará o novo método privado e sua implementação, que no momento está vazia.
![image](https://github.com/user-attachments/assets/b4f70d63-4c95-4098-b123-7aa3f643252f)

![image](https://github.com/user-attachments/assets/5c2f3c87-c776-475a-988d-5373113d68f8)


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

Neste cenário, que passo no final todos os objetos, foi utilizado o recurso field ( numbering : managed ), onde deixamos o framework cuidar da geração dos campos chave - tanto na Root como na child **Items**. Também incremento o campo invreno com + 1 para não repetir o mesmo número, apesar de não ser chave. Este campo seria o nosso número da Invoice. A child Billing utiliza um recurso na hora de salvar (with additional save), que explico em outro momento. Existe um botão **Create Billing from Item** na child Items que tem como objetivo criar itens de Billing relativos a todos os Itens, que explicarei em outro momento.

**O resultado é este**:
![image](https://github.com/user-attachments/assets/d9c303aa-e81c-4fe1-b6fa-d96ed08f2143)


![image](https://github.com/user-attachments/assets/2f1229e8-7abd-4309-9cd0-46fde23c9731)



## composition-child
Abaixo cada objeto para reproduzir o cenário Root-child.

**Tabelas**
```
@EndUserText.label : 'header details'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table yheader {

  key client         : abap.clnt not null;
  key hdrkey         : sysuuid_x16 not null;
  inv_re_no          : abap.char(10) not null;
  ship_loc           : abap.char(25);
  sales_area         : abap.char(25);
  part_no            : abap.char(10);
  ship_status        : abap.char(2);
  uom                : meins;
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  ship_quan          : abap.quan(10,0);
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  reciev_quan        : abap.quan(10,0);
  upd_by             : abap.char(10);
  upd_at             : timestampl;
  locallastchangedat : timestampl;

}

@EndUserText.label : 'Item table'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table yitem {

  key client   : abap.clnt not null;
  @AbapCatalog.foreignKey.screenCheck : false
  key hdrkey   : sysuuid_x16 not null
    with foreign key yheader
      where client = yitem.client
        and hdrkey = yitem.hdrkey;
  key itmkey   : sysuuid_x16 not null;
  inv_re_no    : abap.char(10) not null;
  item_no      : abap.numc(3);
  itm_status   : abap.char(10);
  sales_office : abap.char(10);
  uom          : meins;
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  delv_qty     : abap.quan(10,0);
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  apr_qty      : abap.quan(10,0);

}

@EndUserText.label : 'billing table'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table ybilling {

  key client : abap.clnt not null;
  @AbapCatalog.foreignKey.screenCheck : false
  key hdrkey : sysuuid_x16 not null
    with foreign key yheader
      where client = ybilling.client
        and hdrkey = ybilling.hdrkey;
  key itmkey : sysuuid_x16 not null;
  inv_re_no  : abap.char(10) not null;
  item_no    : abap.numc(3);
  discount   : abap.numc(2);

}

@EndUserText.label : 'header details'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table ytdr_header {

  key client         : abap.clnt not null;
  key hdrkey         : sysuuid_x16 not null;
  invreno            : abap.char(10) not null;
  shiploc            : abap.char(25);
  salesarea          : abap.char(25);
  partno             : abap.char(10);
  shipstatus         : abap.char(2);
  @Semantics.quantity.unitOfMeasure : 'ytdr_header.uom'
  shipquan           : abap.quan(10,0);
  uom                : meins;
  updby              : abap.char(10);
  upd_at             : timestampl;
  locallastchangedat : timestampl;
  "%admin"           : include sych_bdl_draft_admin_inc;

}

@EndUserText.label : 'Item table'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table ytdr_item {

  key client  : abap.clnt not null;
  key hdrkey  : sysuuid_x16 not null;
  key itmkey  : sysuuid_x16 not null;
  invreno     : abap.char(10) not null;
  itemno      : abap.numc(3);
  itmstatus   : abap.char(10);
  salesoffice : abap.char(10);
  uom         : meins;
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  delvqty     : abap.quan(10,0);
  @Semantics.quantity.unitOfMeasure : 'yheader.uom'
  aprqty      : abap.quan(10,0);
  "%admin"    : include sych_bdl_draft_admin_inc;

}

@EndUserText.label : 'billing table'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table ybilling {

  key client : abap.clnt not null;
  @AbapCatalog.foreignKey.screenCheck : false
  key hdrkey : sysuuid_x16 not null
    with foreign key yheader
      where client = ybilling.client
        and hdrkey = ybilling.hdrkey;
  key itmkey : sysuuid_x16 not null;
  inv_re_no  : abap.char(10) not null;
  item_no    : abap.numc(3);
  discount   : abap.numc(2);

}

```

CDSs
```
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'header info'
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
define root view entity yi_header_info
  as select from yheader
  composition [0..*] of yi_item_info    as _Item
  composition [0..*] of yi_billing_info as _billing

{
  key hdrkey             as Hdrkey,
      inv_re_no          as InvReNo,
      ship_loc           as ShipLoc,
      sales_area         as SalesArea,
      part_no            as PartNo,
      ship_status        as ShipStatus,
      uom                as Uom,
      @Semantics.quantity.unitOfMeasure : 'uom'
      ship_quan          as ShipQuan,

      //    reciev_quan as RecievQuan,
      upd_by             as UpdBy,
      @Semantics.systemDateTime.lastChangedAt: true
      upd_at             as upd_at,
      @Semantics.systemDateTime.localInstanceLastChangedAt:true
      locallastchangedat as locallastchangedat,


      _Item,
      _billing
}

@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'item info'
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
define view entity yi_item_info
  as select from yitem
  association to parent yi_header_info as _Header on $projection.Hdrkey = _Header.Hdrkey
{

  key hdrkey       as Hdrkey,
  key itmkey       as Itmkey,
      inv_re_no    as InvReNo,
      item_no      as ItemNo,
      itm_status   as ItmStatus,
      sales_office as SalesOffice,
      uom          as Uom,
      @Semantics.quantity.unitOfMeasure : 'UOM'
      delv_qty     as DelvQty,
      @Semantics.quantity.unitOfMeasure : 'UOM'
      apr_qty      as AprQty,
      _Header

}


@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Billing details'
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
define view entity yi_billing_info
  as select from ybilling
  association to parent yi_header_info as _Header on $projection.Hdrkey = _Header.Hdrkey
{
  key hdrkey    as Hdrkey,
  key itmkey    as Itmkey,
      inv_re_no as InvReNo,
      item_no   as ItemNo,
      discount  as Discount,
      _Header


}


@EndUserText.label: 'Projection of Header Entity'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
define root view entity yc_header_info
  provider contract transactional_query
  as projection on yi_header_info
{


  key     Hdrkey,
          InvReNo,
          ShipLoc,
          SalesArea,
          PartNo,
          ShipStatus,
          Uom,
          ShipQuan,
          UpdBy,
          upd_at,
          locallastchangedat as locallastchangedat
          /* Associations */
          _Item    : redirected to composition child yc_item_info,
          _billing : redirected to composition child yc_billing_info
}


@EndUserText.label: 'Projection of Item Entity'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
define view entity yc_item_info
  as projection on yi_item_info
{
  key Hdrkey,
  key Itmkey,
      InvReNo,
      ItemNo,
      ItmStatus,
      SalesOffice,
      Uom,
      DelvQty,
      AprQty,
      /* Associations */
      _Header : redirected to parent yc_header_info
}


@EndUserText.label: 'billing info'
@AccessControl.authorizationCheck: #NOT_REQUIRED
define view entity yc_billing_info
  as projection on yi_billing_info
{
  key Hdrkey,
  key Itmkey,
      @UI.facet: [ { id:              'Invreno',
                       purpose:         #STANDARD,
                       type:            #IDENTIFICATION_REFERENCE,
                       label:           'Invreno',
                       position:        10 }


                ]
      //     @UI: {
      //          lineItem:       [ { position: 10, importance: #HIGH } ],
      //          identification: [ { position: 10, label: 'InvReNo' } ] }
      //
      @UI: {
            lineItem:       [ { position: 10, importance: #HIGH } ],
                              //{ type: #FOR_ACTION, dataAction: 'YC_HEADER_INFO.createChildFromRoot', label: 'Create From Root' } ],
            identification: [ { position: 10, label: 'InvReNo' } ] }
      InvReNo,
      @UI: {
        lineItem:       [ { position: 20, importance: #HIGH } ],
        identification: [ { position: 20, label: 'ItemNo' } ] }
      ItemNo,
      @UI: {
           lineItem:       [ { position: 30, importance: #HIGH } ],
           identification: [ { position: 30, label: 'Discount' } ] }
      Discount,
      /* Associations */
      _Header : redirected to parent yc_header_info

}

```

**Meta data extensions**
```
@Metadata.layer: #CORE
annotate view yc_header_info with
{
  @UI.facet: [ { id:              'Invreno',
                   purpose:         #STANDARD,
                   type:            #IDENTIFICATION_REFERENCE,
                   label:           'Invreno',
                   position:        10
                   }  ,

            {
            type: #LINEITEM_REFERENCE,
            position: 20,
            label: 'Items',
            targetElement: '_Item'
            },
                          {
            type: #LINEITEM_REFERENCE,
            position: 30,
            label: 'Billing',
            targetElement: '_billing'
            }

            ]
  Hdrkey;

  @UI: {
        lineItem:       [ { position: 11, importance: #HIGH },
                          { type: #FOR_ACTION, dataAction: 'copyInstance', label: 'Copy Sales' } ],
        identification: [ { position: 11, label: 'Inv. Receipt No' } ] }
  @Search.defaultSearchElement: true
  InvReNo;

  @UI: {
        lineItem:       [ { position: 20, importance: #HIGH } ],
        identification: [ { position: 20, label: 'Ship Loc' } ],
        selectionField: [{ position: 10 }] }
  @Search.defaultSearchElement: true
  ShipLoc;

  @UI: {
    lineItem:       [ { position: 30, importance: #HIGH } ],
    identification: [ { position: 30, label: ' Sales Area' } ] }
  @Search.defaultSearchElement: true
  SalesArea;

  @UI: {
    lineItem:       [ { position: 40, importance: #HIGH } ],
    identification: [ { position: 40, label: 'Ship Status' } ] }
  @Search.defaultSearchElement: true
  ShipStatus;

  @UI: {
    lineItem:       [ { position: 50, importance: #HIGH } ],
    identification: [ { position: 50, label: 'Total Quant' } ] }
  ShipQuan;

}

@Metadata.layer: #CUSTOMER
annotate view yc_item_info with
{
  @UI.facet: [ { id:              'Invreno',
                      purpose:         #STANDARD,
                      type:            #IDENTIFICATION_REFERENCE,
                      label:           'Invreno',
                      position:        10 }


               ]

  @UI: {
    lineItem:       [ { position: 10, importance: #HIGH } ],
    identification: [ { position: 10, label: 'ITem No' } ] }


  ItemNo;
  @UI: {
     lineItem:       [ { position: 30, importance: #HIGH } ],
     identification: [ { position: 30, label: 'Status' } ] }
  ItmStatus;
  @UI: {
     lineItem:       [ { position: 40, importance: #HIGH } ],
     identification: [ { position: 40, label: 'Salesoffice' } ] }
  SalesOffice;
  @UI: {
     lineItem:       [ { position: 50, importance: #HIGH } ],
     identification: [ { position: 50, label: 'Uom' } ] }
  Uom;
  @UI: {
     lineItem:       [ { position: 60, importance: #HIGH } ],
     identification: [ { position: 60, label: 'Deliver Quantity' } ] }

  DelvQty;
  @UI: {
     lineItem:       [ { position: 70, importance: #HIGH } ],
     identification: [ { position: 70, label: 'Approved Quantity' } ] }

  AprQty;


}

```

**Behavior definition Root**
```
managed implementation in class ybp_i_header_info unique;
strict( 2 ) ;
with draft;

define behavior for yi_header_info alias Header
persistent table yheader
with additional save
draft table ytdr_header
lock master total etag upd_at
authorization master ( instance )
etag master locallastchangedat

{

    field ( numbering : managed ) Hdrkey;

    // administrative fields: read only
    field ( readonly ) Hdrkey, ShipQuan;

    // instance factory action for copying SO instances
    factory action copyInstance [1];

    // mapping entity's field types with table field types
    mapping for yheader { Hdrkey = hdrkey;
                          InvReNo = inv_re_no;
                          PartNo = part_no;
                          SalesArea = sales_area;
                          ShipLoc = ship_loc;
                          ShipStatus = ship_status;
                          ShipQuan = ship_quan;
                          upd_at = upd_at;
                          locallastchangedat = locallastchangedat ;
                         }

    // standard operations for header entity
    create;
    update;
    delete;

    // associations
    association _Item { create ( features: instance ); with draft; }
    association _billing { create; with draft; }


    draft action Edit;
    draft action Activate;
    draft action Discard;
    draft action Resume;
    draft determine action Prepare;

}


define behavior for yi_item_info alias Item
persistent table yitem
draft table ytdr_item
//with additional save
lock dependent by _Header
authorization dependent by _Header
{
  update;
  delete;
  field ( numbering : managed , readonly )  Itmkey;
  field ( readonly ) hdrkey;
  association _Header { with draft; }

    // mapping entity's field types with table field types
    mapping for yitem { Hdrkey = hdrkey;
                          Itmkey = itmkey;
                          InvReNo = inv_re_no;
                          ItemNo = item_no;
                          ItmStatus =  itm_status;
                          SalesOffice = sales_office;
                          Uom = uom;
                          AprQty = apr_qty;
                          DelvQty = delv_qty;
                      }

}
    define behavior for yi_billing_info alias Billing
    persistent table ybilling
    draft table ytdr_billing
    lock dependent by _Header
    authorization dependent by _Header
{

  update;
  delete;
  field ( numbering : managed , readonly )  Itmkey;
  field ( readonly ) hdrkey;
  association _Header;
  mapping for ybilling{ Hdrkey = hdrkey;
                        Itmkey = itmkey;
                        InvReNo = inv_re_no;
                        ItemNo = item_no ;
                        Discount = discount;
  }

}

```

**Behavior definition Projection**
```
projection;
strict ( 2 );
use draft;
use side effects;

define behavior for yc_header_info alias Header
{
  use create;
  use update;
  use delete;
  use action copyInstance;

  use action Edit;
  use action Activate;
  use action Discard;
  use action Resume;
  use action Prepare;

  use association _Item { create; with draft; }
  use association _billing { create; }

}

define behavior for yc_billing_info alias Billing
{
  use update;
  use delete;

  use association _Header { with draft; }
}

define behavior for yc_item_info alias Item
{
  use update;
  use delete;

  use association _Header;
}

```

**Service Definition**
```
@EndUserText.label: 'Service binding on projection entity'
define service ysd_sales_order {
  expose yc_header_info;
  expose yc_item_info;
  expose yc_billing_info;
}
```


**Classe de implementação do BDEF (Local Type)**
```
CLASS lhc_yi_header_info DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.

    METHODS get_instance_authorizations FOR INSTANCE AUTHORIZATION
      IMPORTING keys REQUEST requested_authorizations FOR yi_header_info RESULT result.
    METHODS copyinstance FOR MODIFY
      IMPORTING keys FOR ACTION header~copyinstance.
    METHODS get_instance_features FOR INSTANCE FEATURES
      IMPORTING keys REQUEST requested_features FOR header RESULT result.

ENDCLASS.

CLASS lhc_yi_header_info IMPLEMENTATION.

  METHOD get_instance_authorizations.
  ENDMETHOD.


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

  METHOD get_instance_features.

    READ ENTITIES OF yi_header_info IN LOCAL MODE
      ENTITY header
         FIELDS ( hdrkey shipstatus )
         WITH CORRESPONDING #( keys )
       RESULT DATA(sales)
       FAILED failed.


    result = VALUE #( FOR sale IN sales
                       ( %tky           = sale-%tky
                         %assoc-_item   = COND #( WHEN sale-shipstatus IS INITIAL
                                                  THEN if_abap_behv=>fc-o-disabled ELSE if_abap_behv=>fc-o-enabled   )
                      ) ).

  ENDMETHOD.

ENDCLASS.

CLASS lsc_zi_header_info DEFINITION INHERITING FROM cl_abap_behavior_saver.

  PROTECTED SECTION.

    METHODS save_modified REDEFINITION.

ENDCLASS.

CLASS lsc_zi_header_info IMPLEMENTATION.

  METHOD save_modified.

    DATA: lit_header  TYPE TABLE OF yheader,
          lit_item    TYPE TABLE OF yitem,
          lit_billing TYPE TABLE OF ybilling,
          wa_billing  TYPE ybilling.

    IF create IS NOT INITIAL.

      lit_header = CORRESPONDING #( create-header MAPPING FROM ENTITY ).
      lit_billing = CORRESPONDING #( lit_header ).
      LOOP AT lit_billing ASSIGNING FIELD-SYMBOL(<fls_billing>).
*        <fls_billing>-inv_re_no = 123456.
        <fls_billing>-item_no   = 100.
        <fls_billing>-discount  = 99.
        MODIFY ybilling FROM TABLE @lit_billing .
      ENDLOOP.

    ENDIF.

    IF update IS NOT INITIAL.

      lit_header = CORRESPONDING #( update-header MAPPING FROM ENTITY ).
      lit_billing = CORRESPONDING #( lit_header ).
      LOOP AT lit_billing ASSIGNING <fls_billing>.
*        <fls_billing>-inv_re_no = 123456.
        <fls_billing>-item_no   = 100.
        <fls_billing>-discount  = 99.
        MODIFY ybilling FROM TABLE @lit_billing .
      ENDLOOP.

    ENDIF.

  ENDMETHOD.

ENDCLASS.
```


**Service binding**
![image](https://github.com/user-attachments/assets/5a2d6503-93cf-4bf3-a817-bb6b61099218)

Diferente do odata v2, para publicar odata V4 é preciso ser feito na transação /IWFND/V4_ADMIN. clicar em preview para abrir o app.

A partir deste cenário é possível implementar alguns outros recursos, alguns deles bem bacanas e saindo do formo, e futuramente pretendo trazer aqui. 

Abraço e até a próxima...
