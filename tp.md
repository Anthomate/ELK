# Partie 1 : Analyse du modèle relationnel

### Explorer la structure de la base de données Northwind
* Voir diagram
## Analyser les relations entre les tables
* Voir diagram
## Analyser les cas d'usage typiques pour la recherche
* Pour les produits :
	* products.product_name
	* products.category_id
	* products.unit_price
	* products.units_in_stock
	* categories.category_name

* Pour les clients :
	* customers.company_name
	* customers.contact_name
	* customers.country, customers.region, customers.city

* Pour les commandes :
	* orders.order_date
	* orders.required_date vs orders.shipped_date
	* orders.ship_country, orders.ship_region
	* order_details.quantity
	* order_details.discount
	
## Définir les objectifs pour l'index Elasticsearch
* Index "products" :
	* product_id & product_name
	* unit_price
	* units_in_stock
	* category_id/name
	* discontinued

* Index "orders"
	* order_id
	* customer_id/company_name
	* Dates : order_date, required_date, shipped_date

# Partie 2 : Conception du mapping Elasticsearch
	
## Déterminer quelles tables doivent être dénormalisées
* Pour l'index "products" :
	* Les informations de "products"
	* Les informations de "categories"
	* Les informations de "suppliers"

* Pour l'index "orders" :
	* Les informations de "orders"
	* Les informations de "customers "
	* Les informations de "order_details"
	* Les informations de "products" → les noms de produits et catégories dans les lignes de commande
## Concevoir la structure des documents
* Structure du document products :
		```json
		{
		  "product_id": 1,
		  "product_name": "Chai",
		  "unit_price": 18.00,
		  "units_in_stock": 39,
		  "units_on_order": 0,
		  "reorder_level": 10,
		  "discontinued": false,
		  "quantity_per_unit": "10 boxes x 20 bags",
		  "category": {
			"category_id": 1,
			"category_name": "Beverages",
			"description": "Soft drinks, coffees, teas, beers, and ales"
		  },
		  "supplier": {
			"supplier_id": 1,
			"company_name": "Exotic Liquids",
			"contact_name": "Charlotte Cooper",
			"contact_title": "Purchasing Manager",
			"country": "UK",
			"region": null,
			"city": "London"
		  }
		}
		```
* Structure du document orders :
		```json
		{
		  "order_id": 10248,
		  "order_date": "1996-07-04T00:00:00.000Z",
		  "required_date": "1996-08-01T00:00:00.000Z",
		  "shipped_date": "1996-07-16T00:00:00.000Z",
		  "freight": 32.38,
		  "ship_via": 3,
		  "ship_name": "Vins et alcools Chevalier",
		  "ship_address": "59 rue de l'Abbaye",
		  "ship_city": "Reims",
		  "ship_region": null,
		  "ship_postal_code": "51100",
		  "ship_country": "France",
		  "customer": {
			"customer_id": "VINET",
			"company_name": "Vins et alcools Chevalier",
			"contact_name": "Paul Henriot",
			"contact_title": "Accounting Manager",
			"country": "France",
			"region": null,
			"city": "Reims"
		  },
		  "employee": {
			"employee_id": 5,
			"last_name": "Buchanan",
			"first_name": "Steven"
		  },
		  "order_details": [
			{
			  "product_id": 11,
			  "unit_price": 14.00,
			  "quantity": 12,
			  "discount": 0,
			  "product_name": "Queso Cabrales",
			  "category_name": "Dairy Products"
			},
			{
			  "product_id": 42,
			  "unit_price": 9.80,
			  "quantity": 10,
			  "discount": 0,
			  "product_name": "Singaporean Hokkien Fried Mee",
			  "category_name": "Grains/Cereals"
			}
		  ]
		}
		```
## Définir les types de données pour chaque champ
* Types de données :
	* Pour les identifiants numériques : integer
	* Pour les noms et textes à rechercher : text avec un champ supplémentaire keyword pour le tri et l'agrégation
	* Pour les enums et valeurs de catégorisation : keyword
	* Pour les prix et valeurs monétaires : float
	* Pour les dates : date
		
## Configurer les analyseurs appropriés
		```json
		"analysis": {
		  "analyzer": {
			"product_name_analyzer": {
			  "type": "custom",
			  "tokenizer": "standard",
			  "filter": [
				"lowercase",
				"asciifolding"
			  ]
			}
		  }
		}
		```
## Créer le mapping dans Elasticsearch
* Mapping pour l'index products :
		```json
		PUT /products
		{
		  "mappings": {
			"properties": {
			  "product_id": { "type": "integer" },
			  "product_name": { 
				"type": "text",
				"analyzer": "product_name_analyzer",
				"fields": {
				  "keyword": { "type": "keyword" }
				}
			  },
			  "unit_price": { "type": "float" },
			  "units_in_stock": { "type": "integer" },
			  "reorder_level": { "type": "integer" },
			  "discontinued": { "type": "boolean" },
			  "units_on_order": { "type": "integer" },
			  "quantity_per_unit": { "type": "text" },
			  "category": {
				"properties": {
				  "category_id": { "type": "integer" },
				  "category_name": { 
					"type": "text",
					"fields": {
					  "keyword": { "type": "keyword" }
					}
				  },
				  "description": { "type": "text" }
				}
			  },
			  "supplier": {
				"properties": {
				  "supplier_id": { "type": "integer" },
				  "company_name": { 
					"type": "text",
					"fields": {
					  "keyword": { "type": "keyword" }
					}
				  },
				  "contact_name": { "type": "text" },
				  "contact_title": { "type": "text" },
				  "country": { "type": "keyword" },
				  "region": { "type": "keyword" },
				  "city": { "type": "keyword" }
				}
			  }
			}
		  },
		  "settings": {
			"analysis": {
			  "analyzer": {
				"product_name_analyzer": {
				  "type": "custom",
				  "tokenizer": "standard",
				  "filter": [
					"lowercase",
					"asciifolding"
				  ]
				}
			  }
			}
		  }
		}
		```
* Mapping pour l'index orders :
		```json
		PUT /orders
		{
		  "mappings": {
			"properties": {
			  "order_id": { "type": "integer" },
			  "order_date": { "type": "date" },
			  "required_date": { "type": "date" },
			  "shipped_date": { "type": "date" },
			  "freight": { "type": "float" },
			  "ship_via": { "type": "integer" },
			  "ship_name": { "type": "text" },
			  "ship_address": { "type": "text" },
			  "ship_city": { "type": "keyword" },
			  "ship_region": { "type": "keyword" },
			  "ship_postal_code": { "type": "keyword" },
			  "ship_country": { "type": "keyword" },
			  "customer": {
				"properties": {
				  "customer_id": { "type": "keyword" },
				  "company_name": { 
					"type": "text",
					"fields": {
					  "keyword": { "type": "keyword" }
					}
				  },
				  "contact_name": { "type": "text" },
				  "contact_title": { "type": "text" },
				  "country": { "type": "keyword" },
				  "region": { "type": "keyword" },
				  "city": { "type": "keyword" }
				}
			  },
			  "employee": {
				"properties": {
				  "employee_id": { "type": "integer" },
				  "last_name": { "type": "text" },
				  "first_name": { "type": "text" }
				}
			  },
			  "order_details": {
				"type": "nested",
				"properties": {
				  "product_id": { "type": "integer" },
				  "unit_price": { "type": "float" },
				  "quantity": { "type": "integer" },
				  "discount": { "type": "float" },
				  "product_name": { "type": "text" },
				  "category_name": { "type": "keyword" }
				}
			  }
			}
		  }
		}
		```
		