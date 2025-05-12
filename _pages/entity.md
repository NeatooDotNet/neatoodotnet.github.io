- It is highly recommended that the concrete be Internal with a public facing Interface
  - This de-couples Entities from Entities for Unit Testing
  - There are components like the constructor and the factory methods that should not be used outside of the library. However, they cannot be private either.


  - Using 'new EntityBaseServices' should be limited to use in Unit Tests