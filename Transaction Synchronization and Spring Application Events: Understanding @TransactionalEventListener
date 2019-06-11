The aim of this article is to explain how  @TransactionalEventListener  works, how it differs from a simple  @EventListener, and finally - what are the threats that we should take into account before using it. Giving a real-life example, I will mainly focus on transaction synchronization issues, not paying too much attention neither to consistency nor application event reliability. A complete SpringBoot project with described examples can be found here.

Example overview
Imagine we have a microservice which manages customers' basic information and triggers activation token generation after a customer is created. From the business perspective, token generation is not an integral part of user creation and should be a separate process (this is a very important assumption, which I will refer to later). To keep things simple, let's assume that a customer looks like this:

@Entity
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    private String token;
    public Customer() {}
    public Customer(String name, String email) {
        this.name = name;
        this.email = email;
    }
    public void activatedWith(String token) {
        this.token = token;
    }
    public boolean hasToken() {
        return !StringUtils.isEmpty(token);
    }
    ...
    //getters
    //equals and hashcode
}


We have a simple spring-data-jpa repository:

public interface CustomerRepository extends JpaRepository<Customer, Long> {}


And below you can see an essence of the business problem - creating a new customer (persisting it in DB) and returning it.

@Service
public class CustomerService {
    private final CustomerRepository customerRepository;
    private final ApplicationEventPublisher applicationEventPublisher;
    public CustomerService(CustomerRepository customerRepository, ApplicationEventPublisher applicationEventPublisher) {
        this.customerRepository = customerRepository;
        this.applicationEventPublisher = applicationEventPublisher;
    }
    @Transactional
    public Customer createCustomer(String name, String email) {
        final Customer newCustomer = customerRepository.save(new Customer(name, email));
        final CustomerCreatedEvent event = new CustomerCreatedEvent(newCustomer);
        applicationEventPublisher.publishEvent(event);
        return newCustomer;
    }
}


As you can see,  CustomerService  depends on two beans:

 CustomerRepository  - interface for the purpose of saving customer
 ApplicationEventPublisher  - Spring's super-interface for  ApplicationContext , which declares the way of event publishing inside Spring application
Please note the constructor injection. If you are not familiar with this technique or not aware of its benefits relative to field injection, please read this article.

Remember to give -1 during the code review if there is no test included! But wait, take it easy, mine is here:

@SpringBootTest
@RunWith(SpringRunner.class)
public class CustomerServiceTest {
  @Autowired
  private CustomerService customerService;
  @Autowired
  private CustomerRepository customerRepository;
  @Test
  public void shouldPersistCustomer() throws Exception {
    //when
    final Customer returnedCustomer = customerService.createCustomer("Matt", "matt@gmail.com");
    //then
    final Customer persistedCustomer = customerRepository.findOne(returnedCustomer.getId());
    assertEquals("matt@gmail.com", returnedCustomer.getEmail());
    assertEquals("Matt", returnedCustomer.getName());
    assertEquals(returnedCustomer, persistedCustomer);
}


The test does one simple thing - it checks whether  createCustomer  method creates a proper customer. One could say that in these kinds of tests I shouldn't pay attention to implementation details (persisting entity through the repository, etc.) and rather put it in some unit test, and I would agree, but let's just leave it to keep the example clearer.

You may ask now, where is the token generation. Well, due to the business case that we are discussing,  createCustomer  method does not seem to be a good place to put any logic apart from simple creation of user (method name should always reflect its responsibility). In that kind of cases it might be a good idea to use the observer pattern to inform interested parties that a particular event took place. Following this considerations, you can see that we are calling  publishEvent  method on  applicationEventPublisher . We are propagating an event of the following type:

public class CustomerCreatedEvent {
  private final Customer customer;
  public CustomerCreatedEvent(Customer customer) {
    this.customer = customer;
  }
  public Customer getCustomer() {
    return customer;
  }
  ...
  //equals and hashCode
}


Note that it is just a POJO. Since Spring 4.2, we are no longer obliged to extend  ApplicationEvent  and can publish any object we like instead. Spring wraps it in  PayloadApplicationEvent  itself.

We do also have an event listener component, like this:

@Component
public class CustomerCreatedEventListener {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomerCreatedEventListener.class);
    private final TokenGenerator tokenGenerator;
    public CustomerCreatedEventListener(TokenGenerator tokenGenerator) {
        this.tokenGenerator = tokenGenerator;
    }
    @EventListener
    public void processCustomerCreatedEvent(CustomerCreatedEvent event) {
        LOGGER.info("Event received: " + event);
        tokenGenerator.generateToken(event.getCustomer());
    }
}


Before we discuss this listener, let's briefly look at the  TokenGenerator  interface:

public interface TokenGenerator {
    void generateToken(Customer customer);
}


and its implementation:

@Service
public class DefaultTokenGenerator implements TokenGenerator {
    private final CustomerRepository customerRepository;
    public DefaultTokenGenerator(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }
    @Override
    public void generateToken(Customer customer) {
        final String token = String.valueOf(customer.hashCode());
        customer.activatedWith(token);
        customerRepository.save(customer);
    }
}


We are simply generating a token, setting it as a customer's property and updating the entity in the database. Good, let's update our test class now, so that it checks not only the customer creation, but also token generation.

@SpringBootTest
@RunWith(SpringRunner.class)
public class CustomerServiceTest {
    @Autowired
    private CustomerService customerService;
    @Autowired
    private CustomerRepository customerRepository;
    @Test
    public void shouldPersistCustomerWithToken() throws Exception {
        //when
        final Customer returnedCustomer = customerService.createCustomer("Matt", "matt@gmail.com");
        //then
        final Customer persistedCustomer = customerRepository.findOne(returnedCustomer.getId());
        assertEquals("matt@gmail.com", returnedCustomer.getEmail());
        assertEquals("Matt", returnedCustomer.getName());
        assertTrue(returnedCustomer.hasToken());
        assertEquals(returnedCustomer, persistedCustomer);
    }
}
@EventListener
As you can see, we have moved token generation logic into a separate component which is good (note the assumption at the beginning of the previous chapter), but do we have a real separation of concerns? Nope.  @EventListener  registers the  processCustomerCreatedEvent  as the listener of  CustomerCreatedEvent , but it is called synchronously within the bounds of the same transaction as  CustomerService . It means, that if something goes wrong with token generation - customer won't be created. Is this the way it should really work? Surely not. Before we generate and set token, we would rather have a customer already created and saved in database (committed). Now this is the time to introduce  @TransactionalEventListenerannotation.

@TransactionalEventListener - transaction synchronization
@TransactionalEventListener  is an  @EventListener  enhanced with the ability to collaborate with the surrounding transaction's phase. We call this a transaction synchronization - in other words, it is a way of registering callback methods to be invoked when the transaction is being completed. Synchronization is possible within the following transaction phases (phase attribute):

AFTER_COMMIT (default setting) - specialization of AFTER_COMPLETION, used when transaction has successfully committed
AFTER_ROLLBACK - specialization of AFTER_COMPLETION, used when transaction has rolled back
AFTER_COMPLETION - used when transaction has completed (regardless the success)
BEFORE_COMMIT - used before transaction commit
When there is no transaction running then the method annotated with  @TransactionalEventListener  won't be executed unless there is a parameter fallbackExecution set to true.

Good, looks like this is something that we are looking for! Let's change  @EventListener  annotation with  @TransactionalEventListener  in  CustomerCreatedEventListener , then:

@TransactionalEventListener
public void processCustomerCreatedEvent(CustomerCreatedEvent event) {
    LOGGER.info("Event received: " + event);
    tokenGenerator.generateToken(event.getCustomer());
}


We need to check now if everything works as we expect - let's run our test:

java.lang.AssertionError: 
Expected :Customer{id=1, name='Matt', email='matt@gmail.com', token='1575323438'}
Actual   :Customer{id=1, name='Matt', email='matt@gmail.com', token='null'}


Why is that? What have we missed? I tell you what, we spent too little time on analyzing how the transaction synchronization works. Now the crucial thing is that we have synchronized token generation with the transaction after it has been committed - so we shouldn't even expect that anything will be committed again! Javadoc for  afterCommit  method of  TransactionSynchronization  interface says it clearly:

The transaction will have been committed already, but the transactional resources might still be active and accessible. As a consequence, any data access code triggered at this point will still "participate" in the original transaction, allowing to perform some cleanup (with no commit following anymore!), unless it explicitly declares that it needs to run in a separate transaction. Hence: Use {@code PROPAGATION_REQUIRES_NEW} for any transactional operation that is called from here.
As we have already stated, we need to have a strong separation of concerns between a service call and an event listener logic. This means that we can use the advice given by Spring authors. Let's try annotating a method inside  DefaultTokenGenerator :

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void generateToken(Customer customer) {
    final String token = String.valueOf(customer.hashCode());
    customer.activatedWith(token);
    customerRepository.save(customer);
}


If we run our test now, it passes!

Caveat: We are discussing interacting with AFTER_COMMIT phase only, but all these considerations apply to all AFTER_COMPLETION phases. In the case of BEFORE_COMMIT phase, none of the above problems should worry you, although you need to make a conscious decision whether your listener's code should really be executed in the same transaction.

Asynchronous execution
What if a token generation is a long lasting process? If it is not an essential part of creating a customer, then we could make one step forward and make our  @TransactionalEventListener-annotated method asynchronous one (via annotating it with  @Async). Asynchronous call means that we will execute listeners  processCustomerCreatedEvent in a separate thread. Bearing in mind that a single transaction in Spring framework is by default thread-bounded, we won't need autonomous transaction in  DefaultTokenGenerator anymore.

Good, let's write a test for this case:

@SpringBootTest
@RunWith(SpringRunner.class)
@ActiveProfiles("async")
public class CustomerServiceAsyncTest {
  @Autowired
  private CustomerService customerService;
  @Autowired
  private CustomerRepository customerRepository;
  @Test
  public void shouldPersistCustomerWithToken() throws Exception {
      //when
      final Customer returnedCustomer = customerService.createCustomer("Matt", "matt@gmail.com");
      //then
      assertEquals("matt@gmail.com", returnedCustomer.getEmail());
      assertEquals("Matt", returnedCustomer.getName());
      //and
      await().atMost(5, SECONDS)
          .until(() -> customerTokenIsPersisted(returnedCustomer.getId()));
  }
  private boolean customerTokenIsPersisted(Long id) {
      final Customer persistedCustomer = customerRepository.findOne(id);
      return persistedCustomer.hasToken();
  }
}


The only thing that differentiates this test from the previous one is that we are using the Awaitility library (a great and powerful tool) in order to await for the async task to complete. We also wrote a simple  customerTokenIsPersisted helper method to check if the token was properly set. And surely test passes brilliantly!

Caveat: I don't recommend performing any async tasks in BEFORE_COMMIT phase as you won't have any guarantee that they will complete before producer's transaction is committed.

Conclusions
@TransactionalEventListener  is a great alternative to  @EventListener  in situations where you need to synchronize with one of the transaction phases. You can declare listeners as synchronous or asynchronous. You need to keep in mind that with the synchronous call you are by default working within the same transaction as the event producer. In case you synchronize to the  AFTER_COMPLETION phase (or one of its specializations), you won't be able to persist anything in the database as there won't be a commit procedure executed anymore. If you need to commit some changes anyway, you can declare an autonomous transaction on the event listener code. BEFORE_COMMIT phase is much simpler because commit will be performed after calling event listeners. With asynchronous calls, you don't have to worry about declaring autonomous transactions, as Spring's transactions are by default thread-bounded (you will get a new transaction anyway). It is a good idea if you have some long running task to be performed. I suggest using asynchronous tasks only when synchronizing to AFTER_COMPLETION phase or one of its specializations. As long as you don't need to have your event listener's method transactional - described problems shouldn't bother you at all.

Further considerations
In a real-life scenario, numerous other requirements might occur. For example, you might need to both persist the customer and send an invitation email like it is depicted below:

@Component
public class CustomerCreatedEventListener {
    private final MailingFacade mailingFacade;
    private final CustomerRepository customerRepository;
    public CustomerCreatedEventListener(MailingFacade mailingFacade, CustomerRepository customerRepository) {
        this.mailingFacade = mailingFacade;
        this.customerRepository = customerRepository;
    }
    @EventListener
    public void processCustomerCreatedEvent(CustomerCreatedEvent event) {
        final Customer customer = event.getCustomer();
        // sending invitation email
        mailingFacade.sendInvitation(customer.getEmail());
    }
}


Imagine a situation when an email is sent successfully, but right after it, our database goes down and it is impossible to commit the transaction (persist customer) - thus, we lose consistency! If such a situation is acceptable within your business use case, then it is completely fine to leave it as it is, but if not then you have a much more complex problem to solve. I intentionally haven't covered consistency and event reliability issues here. If you want to have a broader picture of how you could possibly deal with such situations, I recommend reading this article.
