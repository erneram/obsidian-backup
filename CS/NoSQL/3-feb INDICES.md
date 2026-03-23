Busquedas eficientes ** 
Los indices en mongo utilizan mucha **RAM**
Se debe a que indexa a tipos de STRING
1. Se escanea la coleccción entera
2. documento por documento
3. Al indexarla se hace mas eficiente el scan

Lee rapido pero escribe lento y puede botarlo. 
*Cada indice tiene impacto en desempeño de escritura*
`_id` es el índice default

Casos: 
1. Single Field: Aplicado aun campo -> valor
2. Single Field por objeto embedded: 
3. Compound: Definido para dos o mas campos 


# Gestión

Create: 
	createIndex
View
	getIndexes
Drop
	dropIndex (eliminar)
Hide
	hideIndex (poner una data sin que el manejador sepa que existe, generalmente en performance)


---

## ↩️ Navegación
- [[CS/NoSQL/NoSQL|📦 NoSQL]] → [[CS/CS|💻 CS]] → [[HOME|🏠 HOME]]
