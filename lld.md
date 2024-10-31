<a href="https://javarevisited.blogspot.com/2016/06/design-vending-machine-in-java.html#axzz8q88VOaGA">Desc_1</a>




<a href="https://leetcode.com/discuss/interview-question/object-oriented-design/125218/design-a-vending-machine">Desc_2</a>


```
package vending;

import java.util.List;

/**
 * Public API for Vending Machine
 */
public interface VendingMachine {
    long selectItemAndGetPrice(Product product);
    void insertCoin(Coin coin);
    List<Coin> refund();
    Bucket<Product, List<Coin>> collectItemAndChange();
    void reset();
}


public class VendingMachineImpl implements VendingMachine {
    private Inventory<Coin> cashInventory = new Inventory<>();
    private Inventory<Product> productInventory = new Inventory<>();
    private long totalSales;
    private Product currentProduct;
    private long currentBalance;
    
    private State noCoinInsertedState;
    private State coinInsertedState;
    private State dispensingState;
    private State currentState;

    public VendingMachineImpl() {
        initialize();
    }

    private void initialize() {
        for (Coin c : Coin.values()) {
            cashInventory.put(c, 5);
        }
        
        noCoinInsertedState = new NoCoinInsertedState(this);
        coinInsertedState = new CoinInsertedState(this);
        dispensingState = new DispensingState(this);
        currentState = noCoinInsertedState;
    }
    
    public State getNoCoinInsertedState() {
        return noCoinInsertedState;
    }

    public State getCoinInsertedState() {
        return coinInsertedState;
    }

    public State getDispensingState() {
        return dispensingState;
    }

    public void setState(State state) {
        currentState = state;
    }

    @Override
    public long selectItemAndGetPrice(Product product) {
        if (productInventory.hasItem(product)) {
            currentProduct = product;
            return currentProduct.getPrice();
        }
        throw new SoldOutException("Sold Out, Please buy another product.");
    }

    @Override
    public void insertCoin(Coin coin) {
        currentState.insertCoin(coin);
    }

    @Override
    public Bucket<Product, List<Coin>> collectItemAndChange() {
        Product product = currentState.dispense();
        List<Coin> change = currentState.collectChange();
        return new Bucket<>(product, change);
    }

    @Override
    public List<Coin> refund() {
        return currentState.refund();
    }

    @Override
    public void reset() {
        cashInventory.clear();
        productInventory.clear();
        totalSales = 0;
        currentBalance = 0;
        currentState = noCoinInsertedState;
    }

    public void addBalance(long amount) {
        currentBalance += amount;
    }

    public long getBalance() {
        return currentBalance;
    }

    public Product getCurrentProduct() {
        return currentProduct;
    }

    public void updateSales(long amount) {
        totalSales += amount;
    }

    private List<Coin> getChange(long amount) throws NotSufficientChangeException {
    List<Coin> changes = new ArrayList<>();
    Inventory changeInventory = new Inventory();
    
    if (amount > 0) {
        long balance = amount;
        List<Coin> inventory = cashInventory.getAll();
        
        Collections.sort(inventory, new Comparator<Coin>() {
            @Override
            public int compare(Coin o1, Coin o2) {
                return o2.getDenomination() - o1.getDenomination();
            }
        });
        
        while (balance > 0) {
            boolean isContinue = false;
            for (int i = 0; i < inventory.size(); i++) {
                if (balance >= inventory.get(i).getDenomination()
                        && cashInventory.hasItem(inventory.get(i), changeInventory.getQuantity(inventory.get(i)) + 1)) {
                    balance -= inventory.get(i).getDenomination();
                    changeInventory.add(inventory.get(i));
                    isContinue = true;
                    break;
                }
            }
            if (!isContinue) {
                break;
            }
        }

        if (balance != 0) {
            throw new NotSufficientChangeException("Not sufficient change, please try another product!");
        }

        for (Coin c : changeInventory.getAll()) {
            for (int i = 0; i < changeInventory.getQuantity(c); i++) {
                changes.add(c);
            }
        }
    }

    return changes;
}

}


public interface State {
    void insertCoin(Coin coin);
    Product dispense();
    List<Coin> collectChange();
    List<Coin> refund();
}



public class NoCoinInsertedState implements State {
    private final VendingMachineImpl vendingMachine;

    public NoCoinInsertedState(VendingMachineImpl vendingMachine) {
        this.vendingMachine = vendingMachine;
    }

    @Override
    public void insertCoin(Coin coin) {
        vendingMachine.addBalance(coin.getDenomination());
        vendingMachine.setState(vendingMachine.getCoinInsertedState());
    }

    @Override
    public Product dispense() {
        throw new MachineException("No coin inserted");
    }

    @Override
    public List<Coin> collectChange() {
        throw new MachineException("No coin inserted");
    }

    @Override
    public List<Coin> refund() {
        throw new MachineException("No coin inserted");
    }
}


class CoinInsertedState implements State {
    private final VendingMachineImpl vendingMachine;

    public CoinInsertedState(VendingMachineImpl vendingMachine) {
        this.vendingMachine = vendingMachine;
    }

    @Override
    public void insertCoin(Coin coin) {
        throw new MachineException("Coin already inserted");
    }

    @Override
    public Product dispense() {
        if (vendingMachine.getBalance() >= vendingMachine.getCurrentProduct().getPrice()) {
            vendingMachine.setState(vendingMachine.getDispensingState());
            return vendingMachine.getCurrentProduct();
        }
        throw new MachineException("Not enough balance");
    }

    @Override
    public List<Coin> collectChange() {
        throw new MachineException("Product not dispensed yet");
    }

    @Override
    public List<Coin> refund() {
        List<Coin> refund = vendingMachine.getChange(vendingMachine.getBalance());
        vendingMachine.reset();
        return refund;
    }
}

class DispensingState implements State {
    private final VendingMachineImpl vendingMachine;

    public DispensingState(VendingMachineImpl vendingMachine) {
        this.vendingMachine = vendingMachine;
    }

    @Override
    public void insertCoin(Coin coin) {
        throw new MachineException("Please wait, dispensing in progress");
    }

    @Override
    public Product dispense() {
        Product product = vendingMachine.getCurrentProduct();
        vendingMachine.updateSales(product.getPrice());
        vendingMachine.getProductInventory().deduct(product);
        vendingMachine.setState(vendingMachine.getNoCoinInsertedState());
        return product;
    }

    @Override
    public List<Coin> collectChange() {
        long changeAmount = vendingMachine.getBalance() - vendingMachine.getCurrentProduct().getPrice();
        List<Coin> change = vendingMachine.getChange(changeAmount);
        vendingMachine.reset();
        return change;
    }

    @Override
    public List<Coin> refund() {
        throw new MachineException("Dispensing in progress, cannot refund");
    }
}



class VendingMachineFactory {  
    
    public static VendingMachine createVendingMachine() {
        VendingMachineImpl vendingMachine = new VendingMachineImpl();
        // Setting the initial state for the vending machine
        VendingMachineState initialState = new InitialState(vendingMachine);
        vendingMachine.setState(initialState);
        return vendingMachine;
    }

    public static VendingMachine createReadyVendingMachine() {
        VendingMachineImpl vendingMachine = new VendingMachineImpl();
        // Setting ready state for the vending machine
        VendingMachineState readyState = new ReadyState(vendingMachine);
        vendingMachine.setState(readyState);
        return vendingMachine;
    }
}


class Main {

    public static void main(String[] args) {
        // Create a new vending machine instance using the factory
        VendingMachine vendingMachine = VendingMachineFactory.createVendingMachine();

        // Display initial vending machine status
        System.out.println("Vending Machine created and initialized.");
        vendingMachine.printStats();

        try {
            // Step 1: Select an item and get its price
            Item item = new Item("Coke", 101, 25); // item with name, ID, and price
            long price = vendingMachine.selectItemAndGetPrice(item);
            System.out.println("Selected item: " + item.getName() + ", Price: " + price);

            // Step 2: Insert coins until enough balance is reached
            vendingMachine.insertCoin(Coin.QUARTER);
            vendingMachine.insertCoin(Coin.QUARTER); // assuming each quarter is 25
            System.out.println("Current balance after inserting coins: 50");

            // Step 3: Dispense the item and collect change
            Bucket<Item, List<Coin>> itemAndChange = vendingMachine.collectItemAndChange();
            System.out.println("Collected item: " + itemAndChange.getFirst().getName());
            System.out.println("Change received: ");
            for (Coin coin : itemAndChange.getSecond()) {
                System.out.print(coin + " ");
            }
            System.out.println();

            // Step 4: Print final status of the vending machine
            vendingMachine.printStats();
            
        } catch (SoldOutException | NotFullPaidException | NotSufficientChangeException e) {
            System.out.println("Error: " + e.getMessage());
        } finally {
            // Reset the vending machine for future transactions
            vendingMachine.reset();
            System.out.println("Vending machine has been reset.");
        }
    }
}
```





# Class Descriptions
VendingMachine (Context): The main class representing the vending machine, responsible for maintaining balance, tracking sales, storing items, and handling the machine's current state. It uses different states to control interactions.

VendingMachineState (State Interface): This interface defines the methods for various actions (insertCoin, selectItem, dispense), implemented differently depending on the current machine state.

NoCoinInsertedState, CoinInsertedState, DispensingState: These are the concrete states that implement VendingMachineState. They represent distinct behaviors for each machine state:

NoCoinInsertedState: Allows coin insertion but prevents item selection or dispensing.
CoinInsertedState: Allows item selection, but attempting to insert more coins throws an exception.
DispensingState: Manages item dispensing and prevents new actions until the dispensing process is complete.
Inventory: Manages inventory for both items and coins. It provides methods like add, deduct, and hasItem to maintain quantities.

VendingMachineFactory: A factory class for creating instances of VendingMachine. Useful for adding different configurations for other machine types.

Coin (Enum): Represents the various coin denominations available in the vending machine. Each coin has a fixed denomination.

Item: Represents a product with attributes like name, price, and id, unlike the enum used previously.

Explaining the Design in an LLD Interview
Overview: Begin by explaining the high-level functionality of the vending machine and how your design addresses various requirements: selecting items, inserting coins, dispensing items, and handling exceptions (e.g., insufficient funds, no stock).

# Key Design Patterns:


State Pattern: Explain that the State Pattern enables the vending machine to alter its behavior based on its state (e.g., NoCoinInsertedState, CoinInsertedState, DispensingState).
               This pattern encapsulates each state’s behavior in a class, allowing flexible state transitions and method handling.

Factory Pattern: Describe how the Factory Pattern is used for creating vending machine instances. 
               The factory allows the creation of different configurations, helping with future scalability if new types of vending machines need to be added.
               
# Class Details:

VendingMachine: Explain that this class serves as the main context, coordinating inventory and balance and delegating actions to different states.

VendingMachineState Interface and Concrete States: Outline the methods each state implements differently. For instance:

In NoCoinInsertedState, the insertCoin method updates the balance and transitions to CoinInsertedState.

In CoinInsertedState, pressButton checks product availability, then moves to DispensingState.

In DispensingState, dispense completes the transaction and reverts the machine back to NoCoinInsertedState.

Error Handling: Describe the custom exceptions (e.g., SoldOutException, NotFullPaidException, NotSufficientChangeException) and how they ensure robust error handling in cases like sold-out items, insufficient balance, or a lack of change.

# Design Flexibility and Extensibility:

With the State Pattern, adding new states (like OutOfServiceState) is straightforward.

The Factory Pattern allows adding other types of vending machines without altering core functionality.

Possible Extensions: End with ideas for enhancing the design, such as adding new product categories, implementing a user interface, or integrating payment APIs.
