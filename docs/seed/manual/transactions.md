---
title: "Transactions"
type: "home"
zones:
    - "Seed"
sections:
    - "SeedManual"
menu:
    SeedManual:
        weight: 60
---

Seed transaction management allows interactions between the application code and one or more external resource(s) to be
done transactionally (ie. in an all-or-nothing paradigm). It is used in conjunction with other supports handling external 
resources such as persistence or messaging. For more detail about transactions, refer to this [wikipedia page](http://en.wikipedia.org/wiki/Transaction_processing).

# Dependency

To enable transactions in your project, add the `seed-transaction` module to your classpath. 

{{< dependency "org.seedstack.seed" "seed-transaction" >}}

Note that this dependency is rarely explicitly required as it is transitively provided by any transaction-capable add-on
like JPA persistence, JMS, ...

# Transaction manager

The transaction manager is responsible for detecting transaction boundaries in the application code and wrap them with
an interceptor. This interceptor will then automatically create, commit or rollback and release a transaction when the
transactional code is invoked. The behavior of the transaction manager is heavily customizable depending on your business requirement.

## Configuration

A Seed application can have **only one** transaction manager. The transaction manager is specified with following configuration property:

	org.seedstack.seed.transaction.transaction-manager = fully.qualified.name.of.TransactionManagerClass

## Local transaction manager

The local transaction manager manages transactions within the application. It cannot handle global transactions managed 
by an external transaction monitor like a J2EE Web server and doesn't support spanning transactions over multiple 
resources. However it is very lightweight and adequate for most common applications uses. This is the default transaction
manager.

	org.seedstack.seed.transaction.transaction-manager = org.seedstack.seed.transaction.internal.LocalTransactionManager

## JTA transaction manager	

The JTA transaction manager integrates code demarcated with Seed transactions with any external JTA-compliant transaction
monitor such as ones found in J2EE Web servers. To use it, just specify the following configuration property:

	org.seedstack.seed.transaction.transaction-manager = org.seedstack.seed.transaction.internal.JtaTransactionManager
	
Some supports may need additional configuration to be able to participate in a JTA transaction. 

# Transaction metadata

Transactions have several attributes that define their behavior and outcome:

* **Propagation** determines if a new transaction should be started and/or how an existing transaction should be handled, if any. Possible values are:
	* `REQUIRED`: use existing transaction or create a new one if none exists. **It is the default value.**
	* `REQUIRES_NEW`: create a new transaction and suspend the previous one if any exists.
	* `MANDATORY`: throw an exception if no existing transaction is found.
	* `SUPPORTS`: execute code outside any transaction if no existing transaction is found.
	* `NOT_SUPPORTED`: execute code outside any transaction and suspend the current transaction if one exists.
	* `NEVER`: execute code outside any transaction and throw an exception if an existing transaction is found.
* **Rollback** behavior is defined:
	* a list of exception classes triggering a rollback if thrown from transactional code (default is `java.lang.Exception`)
	* a list of exception classes that will NOT trigger a rollback if thrown from transactional code (default value is empty).
* **readOnly** attribute determines if a transaction is read-only or not.
* **rollbackOnParticipationFailure** attribute determines if a participating method should mark the transaction as rollback-only if an error occurs.
* The **transaction handler** will interact with the transacted resource(s).
* **Resource identifier** is required when multiple resources are handled by the same transaction handler.

These attributes are called transaction metadata and default values can be overridden explicitly or by inferred
values from transaction metadata resolvers. **The explicitly specified values always take precedence over the automatically
inferred ones, which in turn always take precedence over the default ones.**

# Demarcation via annotation

Transaction metadata can be explicitly specified through {{< java "org.seedstack.seed.transaction.Transactional" "@" >}} annotation and associated annotations (for
each type of transactional resource). These annotations can be placed on methods, classes, interfaces and other annotations.
The search for the {{< java "org.seedstack.seed.transaction.Transactional" "@" >}} annotation starts for all methods of any Seed managed class, using the following order:

* The method and any annotation on this method,
* The declaring class and any annotation on this class,
* Any superclass or interface up in the hierarchy, with the following sub-order for each class/interface:
	* The overridden method and any annotation on this method,
	* The class/interface and any annotation on this class.
	
If no {{< java "org.seedstack.seed.transaction.Transactional" "@" >}} annotation is found when the top of the class hierarchy is reached, the method is not transactional
and not intercepted at all.

# Automatic metadata resolution

Transaction metadata resolvers are means of automatically determining transaction metadata for a specific context. A
transaction metadata resolver must implement {{< java "org.seedstack.seed.transaction.spi.TransactionMetadataResolver" >}} interface 
and provide a default constructor. They are scanned and registered at application startup and queried **in no predefined
order** at the beginning of method interception. Since they are queried inside the transaction interceptor,
**an explicit transaction demarcation has to be present** in the first place. They cannot add behavior to not
demarcated code.

## Resolving

The `resolve()` method is called on each resolver with the intercepted method as parameter. Its return is Return is either an instance 
of {{< java "org.seedstack.seed.transaction.spi.TransactionMetadata" >}} with the inferred attributes set ot `null` when nothing can be inferred.
inferred attributes set.

Seed provides a built-in always active resolver which automatically associate the transaction handler if only
one is available. In this case, it is not necessary to explicitly specify the corresponding transaction handler.

Other Seed modules can register their own resolvers that will infer more transaction metadata for specific contexts. For
instance the JMS add-on can automatically detect the transacted resource when the {{< java "org.seedstack.seed.transaction.Transactional" "@" >}} annotation is used
inside a JMS listener. More documentation is available in modules where this is applicable.

Remember that an explicitly specified {{< java "org.seedstack.seed.transaction.Transactional" "@" >}} annotation will always override any automatically resolved metadata.

# Examples

## With an explicit JMS resource

	@Transactional
    @JmsConnection("connection1")
    public void send(String stringMessage) throws JMSException {
        Destination queue = session.createQueue("queue1");
        TextMessage message1 = session.createTextMessage();
        message1.setText(stringMessage);
        MessageProducer producer = session.createProducer(queue);
        producer.send(message1);
    }
	
## With an explicit JPA resource 

	@Inject
    Item1Repository item1Repository;
	
    @Transactional
    @JpaUnit("unit1")
    public void save() throws Exception {
        Item1 item1 = new Item1();
        item1.setName("item1Name");
        item1Repository.save(item1);
    }
	
## With an implicit resource


	@Inject
	Item1Repository item1Repository;
	
	@Transactional
	public void save() throws Exception {
		Item1 item1 = new Item1();
		item1.setName("item1Name");
		item1Repository.save(item1);
	}
	
	
 * JPA resource is assumed since it is the only resource available.
 * JpaUnit is assumed since only one is defined.
 