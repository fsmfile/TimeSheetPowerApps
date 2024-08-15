Abaixo, organizei a documentação do seu aplicativo TimeSheet no Power Apps, separando cada controle e código de forma mais clara:

---

### **Configuração Geral do App**

- **App**

(Evento: OnStart)
  ```plaintext
  Set(
      varLogin;
      Left(User().Email; Find("@"; User().Email) - 1)
  );;
  Set(varNewRecord; false);;
  Set(SelectedCboSubCategoria; Blank());;
  ClearCollect(
      ColCategoriasConcat;
      ForAll(
          tbl_cad_categoriaTimeSheet;
          {
              ID: ID;
              Title: Title;
              SubCategoria: SubCategoria;
              CategoriaConcatenada: ID_Ctg & " :: " & Title & " :: " & SubCategoria
          }
      )
  )
  ```

---

### **Telas**

#### **TimeSheet**

- **Cabeçalho** 
  - Título: `"Seja bem vindo, " & User().FullName & "!"`

##### **Contêiner: Ctn_Preenchidos**

- **Galeria: LstPreenchidos**
  - **Items:**
    ```plaintext
    With(
        {
            startDate: DateValue("01/" & Text(Month(CxDtAtividade.SelectedDate); "00") & "/" & Text(Year(CxDtAtividade.SelectedDate); "0000"));
            endDate: DateAdd(DateValue("01/" & Text(Month(CxDtAtividade.SelectedDate); "00") & "/" & Text(Year(CxDtAtividade.SelectedDate); "0000")); 1; TimeUnit.Months)
        };
        AddColumns(
            ForAll(
                Sequence(Day(endDate - startDate); 1; 1);
                {
                    Dia: DateAdd(startDate; Value - 1; "Days");
                    DiaSemana: Left(Text(DateAdd(startDate; Value - 1; "Days"); "dddd"); 3)
                }
            );
            HorasTrab;
            With(
                {
                    TotalHoras: Sum(
                        Filter(tbl_TimeSheet; DateValue(Text(DtAtividade; "dd/mm/yyyy")) = Dia);
                        Value(Left(HorasTrab; 2)) * 60 + Value(Mid(HorasTrab; 4; 2))
                    )
                };
                Text(Int(TotalHoras / 60); "00") & ":" & Text(Mod(TotalHoras; 60); "00")
            )
        )
    )
    ```

- **Dentro da Galeria LstPreenchidos:**
  - **btnSetaEsq (OnSelect):**
    ```plaintext
    UpdateContext({SelectedDate: ThisItem.Dia});;
    Set(varSelectedDate; ThisItem.Dia);;
    CxDtAtividade.SelectedDate = varSelectedDate;;
    Set(varNewRecord; true);;
    ClearCollect(varEmptyRecord; {DtAtividade: Blank(); Categoria: Blank(); Funcionario: Blank(); HorasTrab: Blank(); Descricao: Blank()});;
    Set(varSelectedItem; First(varEmptyRecord));;
    Reset(DataAtividade);;
    Reset(CboCategoria);;
    Set(SelectedCboSubCategoria; Blank());;
    Reset(CboSubCategoria);;
    Reset(txtHorasTrab);;
    Reset(txtObservacao)
    ```

##### **Contêiner: Ctn_Formulario**

- **Campo: SelectDate (DataAtividade)**
  - **DefaultDate:**
    ```plaintext
    If(IsBlank(varSelectedDate); Today(); varSelectedDate)
    ```

- **Caixa de Combinação: CboCategoria**
  - **OnChange:** `Reset(CboSubCategoria)`
  - **Items:**
    ```plaintext
    Distinct(tbl_cad_categoriaTimeSheet; Título)
    ```
  - **DefaultSelectedItems:**
    ```plaintext
    If(varNewRecord; Blank(); If(varSelectFromFavorites; {Value: varSelectedCategoria.Título}; {Value: varSelectedCategoria.Título}))
    ```
  
- **Caixa de Combinação: CboSubCategoria**
  - **DefaultSelectedItems:**
    ```plaintext
    If(varNewRecord; Blank(); If(varSelectFromFavorites; {Value: varSelectedCategoria.SubCategoria}; {Value: varSelectedCategoria.SubCategoria}))
    ```
  - **Items:**
    ```plaintext
    Distinct(Filter(tbl_cad_categoriaTimeSheet; Título = CboCategoria.Selected.Value); SubCategoria)
    ```

- **Caixa de Texto: txtHorasTrab**
  - **Default:**
    ```plaintext
    If(IsBlank(varSelectedItem); ""; varSelectedItem.HorasTrab)
    ```

- **Caixa de Texto: txtObservacao**
  - **Default:**
    ```plaintext
    If(IsBlank(varSelectedItem); ""; varSelectedItem.Observacao)
    ```

- **Botão Novo Registro: btnNovoReg (OnSelect)**
  ```plaintext
  Set(varNewRecord; true);;
  ClearCollect(varEmptyRecord; {DtAtividade: Blank(); Categoria: Blank(); Funcionario: Blank(); HorasTrab: Blank(); Descricao: Blank()});;
  Set(varSelectedItem; First(varEmptyRecord));;
  Reset(DataAtividade);;
  Reset(CboCategoria);;
  Set(SelectedCboSubCategoria; Blank());;
  Reset(CboSubCategoria);;
  Reset(txtHorasTrab);;
  Reset(txtObservacao)
  ```

- **Botão Cancelar: btnCancelar (OnSelect)**
  ```plaintext
  ClearCollect(
      [Ctn_Formulario];
      {
          DtAtividade: Blank();
          CboCategoria: Blank();
          CboSubCategoria: Blank();
          txtHoras: "";
          txtObservacao: ""
      }
  );;
  Reset(CboCategoria);;
  Reset(CboSubCategoria);;
  Reset(txtHorasTrab);;
  Reset(txtObservacao);;
  Reset(DataAtividade);;
  Set(varSelectFromFavorites; false)
  ```

- **Botão Salvar: btnSalvar (OnSelect)**
  ```plaintext
  If(
      Not(IsBlank(DataAtividade.SelectedDate)) && 
      Not(IsBlank(CboSubCategoria.Selected.Value)) && 
      Not(IsBlank(txtHorasTrab.Text));
      
      If(
          varNewRecord;
          // Código para criar um novo registro
          Patch(
              tbl_TimeSheet;
              Defaults(tbl_TimeSheet);
              {
                  DtAtividade: DataAtividade.SelectedDate;
                  Categoria: {
                      '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                      Id: Value(LookUp(tbl_cad_categoriaTimeSheet; SubCategoria = CboSubCategoria.Selected.Value).ID);
                      Value: CboSubCategoria.Selected.Value
                  };
                  Funcionario: {
                      '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                      Id: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).ID;
                      Value: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).login_usuarioSistema
                  };
                  HorasTrab: txtHorasTrab.Text;
                  Descricao: txtObservacao.Text
              }
          );
          
          // Código para editar um registro existente
          Patch(
              tbl_TimeSheet;
              LookUp(tbl_TimeSheet; ID = varSelectedItem.ID); // Buscar o item correto para edição
              {
                  DtAtividade: DataAtividade.SelectedDate;
                  Categoria: {
                      '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                      Id: Value(LookUp(tbl_cad_categoriaTimeSheet; SubCategoria = CboSubCategoria.Selected.Value).ID);
                      Value: CboSubCategoria.Selected.Value
                  };
                  Funcionario: {
                      '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                      Id: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).ID;
                      Value: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).login_usuarioSistema
                  };
                  HorasTrab: txtHorasTrab.Text;
                  Descricao: txtObservacao.Text
              }
          )
      );
  
      Set(varNewRecord; false);;
      Reset(DataAtividade);;
      Reset(CboCategoria);;
      Set(SelectedCboSubCategoria; Blank());;
      Reset(CboSubCategoria);;
      Reset(txtHorasTrab);;
      Reset(txtObservacao);;
      
      // Força a atualização da galeria após o Patch
      Refresh(tbl_TimeSheet);;
      Notify("Salvo com Sucesso"; NotificationType.Success)
  ;;
      Notify("Erro: Verifique se todos os campos estão preenchidos corretamente."; NotificationType.Error)
  );;
  Set(varSelectFromFavorites; false);;
  Set(varNewRecord; true)
  ```

---

#### **Contêiner: Ctn_ListaCategorias**

- **Campo de Data: CxDtAtividade**
  - **DefaultDate:** `SelectedDate`
  - **OnChange:**
    ```plaintext
    If(
        CxDtAtividade.SelectedDate > Today();
        Notify("A data não pode ser maior que a data atual."; NotificationType.Error);;
        UpdateContext({SelectedDate: Today()});;
        Reset(CxDtAtividade);;
        UpdateContext({SelectedDate: CxDtAtividade.SelectedDate})
    );;
    Set(varSelectedDate; CxDtAtividade.SelectedDate)
    ```

- **Galeria: LstCategoriasHoras**
  - **Items:**
    ```plaintext
    Filter(tbl_TimeSheet; Funcionario.Value = Text(varLogin) && DtAtividade = CxDtAtividade.SelectedDate)
    ```

- **Dentro da Galeria L

stCategoriasHoras:**
  - **Subtitle3 (Text):**
    ```plaintext
    LookUp(tbl_cad_categoriaTimeSheet; ID = ThisItem.Categoria.Id; ID_Ctg)
    & " :: " & LookUp(tbl_cad_categoriaTimeSheet; ID = ThisItem.Categoria.Id; Título)
    ```

  - **Title3 (Text):** `ThisItem.HorasTrab`
  
  - **Botão (Icon1 OnSelect):** `Remove(tbl_TimeSheet;ThisItem)`

  - **Botão (btnSetaDir_LstPreenchidos OnSelect):**
    ```plaintext
    Set(varSelectedItem; ThisItem);;
    Set(varNewRecord; false);;
    Set(varSelectFromFavorites;false);;
    Set(varSelectedCategoria; LookUp(tbl_cad_categoriaTimeSheet; ID = LstCategoriasHoras.Selected.Categoria.Id));;
    Set(SelectedCboSubCategoria; {Value: varSelectedCategoria.SubCategoria});;
    UpdateContext({contextHorasTrab: ThisItem.HorasTrab});;
    UpdateContext({contextObservacao: ThisItem.Descricao})
    ```

---

#### **Contêiner: CtnCategoriaFavorita**

- **Botão adicionar categoria favorita: BtnAdicionarCatFav (OnSelect)**

- **Galeria: LstCategoriasFavoritas**
  - **Items:**
    ```plaintext
    Filter(tbl_CategoriaFavorita_TimeSheet; Usuario.Value = varLogin)
    ```

- **Dentro da Galeria LstCategoriasFavoritas:**
  - **Botão adicionar item da galeria: btn_add_LstCatFav (OnSelect)**
    ```plaintext
    Set(varSelectedCategoria; LookUp(tbl_cad_categoriaTimeSheet; ID = LstCategoriasFavoritas.Selected.Categoria.Id));;
    Set(SelectedCboSubCategoria; {Value: varSelectedCategoria.SubCategoria});;
    Set(varSelectFromFavorites; true);;
    Set(varNewRecord; false)
    ```

  - **Botão excluir item da galeria: btn_excluir_LstCatFav (OnSelect)** `Remove(tbl_CategoriaFavorita_TimeSheet; ThisItem)`

  - **Texto1 (txt_ID_LstCatFav):** `ThisItem.Categoria.Value`

  - **Texto2 (txt_SubCategoria_LstCatFav):**
    ```plaintext
    LookUp(tbl_cad_categoriaTimeSheet; ID_Ctg = ThisItem.Categoria.Value;SubCategoria)
    ```

---

#### **Contêiner: Ctn_Form_CategoriasFav**

- **Label com o título da janela: label_Titulo_CadCatFav**

- **Label para o campo de buscar categoria: label_Buscar_CadCatFav**

- **Entrada de texto para Filtrar enquanto digita uma categoria na Caixa de Listagem: txt_buscaCatFav**

- **Caixa de Listagem: LstCategorias_CadCatFav**
  - **Items:** `ColCategoriasConcat`
  - **OnSelect:**
    ```plaintext
    If(
        CountRows(
            Filter(tbl_CategoriaFavorita_TimeSheet; Usuario.Id = LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).ID && Categoria.Id = LstCategorias_CadCatFav.Selected.ID)
        ) = 0;
        Patch(
            tbl_CategoriaFavorita_TimeSheet;
            Defaults(tbl_CategoriaFavorita_TimeSheet);
            {
                Categoria: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                    Id: LookUp(tbl_cad_categoriaTimeSheet; ID = LstCategorias_CadCatFav.Selected.ID).ID;
                    Value: LookUp(tbl_cad_categoriaTimeSheet; ID = LstCategorias_CadCatFav.Selected.ID).ID_Ctg
                };
                Usuario: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedReference";
                    Id: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).ID;
                    Value: LookUp(tbl_cad_UsuarioSistema; login_usuarioSistema = varLogin).login_usuarioSistema
                }
            }
        );
        Notify("Categoria favorita adicionada com sucesso!"; NotificationType.Success)
    )
    ```

---