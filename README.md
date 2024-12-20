live: https://bookheaven-om.vercel.app/


Creating a full-stack project that integrates Stripe payments, TypeScript, React, PostgreSQL, and TypeORM involves multiple steps. Below, I'll walk you through the process:


---

Backend (TypeScript)

Folder Structure

src/
├── controllers/
│   ├── user.controller.ts
│   ├── payment.controller.ts
├── services/
│   ├── user.service.ts
│   ├── payment.service.ts
├── routes/
│   ├── user.routes.ts
│   ├── payment.routes.ts
├── entities/
│   ├── User.ts
├── database/
│   ├── connection.ts
├── server.ts

1. Setup TypeORM Entity (User)

// src/entities/User.ts
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";

@Entity()
export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column()
    email: string;

    @Column()
    username: string;

    @Column()
    password: string;

    @Column()
    pinCode: string;

    @Column()
    country: string;

    @Column()
    address: string;
}


---

2. Database Connection

// src/database/connection.ts
import { DataSource } from "typeorm";
import { User } from "../entities/User";

export const AppDataSource = new DataSource({
    type: "postgres",
    host: "localhost",
    port: 5432,
    username: "your-username",
    password: "your-password",
    database: "your-database",
    entities: [User],
    synchronize: true, // Do not use in production!
});


---

3. Stripe Service

// src/services/payment.service.ts
import Stripe from "stripe";

export class PaymentService {
    private stripe: Stripe;

    constructor() {
        this.stripe = new Stripe("your-stripe-secret-key", { apiVersion: "2022-11-15" });
    }

    async createPaymentIntent(priceId: string, quantity: number, customerDetails: any) {
        const paymentIntent = await this.stripe.paymentIntents.create({
            amount: quantity * 100, // Stripe requires amount in cents
            currency: "usd",
            description: `Payment for Price ID: ${priceId}`,
            metadata: customerDetails,
        });
        return paymentIntent;
    }
}


---

4. User Service

// src/services/user.service.ts
import { AppDataSource } from "../database/connection";
import { User } from "../entities/User";

export class UserService {
    private userRepository = AppDataSource.getRepository(User);

    async createUser(userData: Partial<User>) {
        const newUser = this.userRepository.create(userData);
        return await this.userRepository.save(newUser);
    }
}


---

5. Payment Controller

// src/controllers/payment.controller.ts
import { Request, Response } from "express";
import { PaymentService } from "../services/payment.service";

export class PaymentController {
    private paymentService = new PaymentService();

    async handlePayment(req: Request, res: Response) {
        try {
            const { priceId, quantity, customerDetails } = req.body;
            const paymentIntent = await this.paymentService.createPaymentIntent(priceId, quantity, customerDetails);
            res.status(200).json({ success: true, paymentIntent });
        } catch (error) {
            res.status(500).json({ success: false, error: error.message });
        }
    }
}


---

6. User Controller

// src/controllers/user.controller.ts
import { Request, Response } from "express";
import { UserService } from "../services/user.service";

export class UserController {
    private userService = new UserService();

    async registerUser(req: Request, res: Response) {
        try {
            const userData = req.body;
            const user = await this.userService.createUser(userData);
            res.status(201).json({ success: true, user });
        } catch (error) {
            res.status(500).json({ success: false, error: error.message });
        }
    }
}


---

7. Routes

// src/routes/user.routes.ts
import { Router } from "express";
import { UserController } from "../controllers/user.controller";

const router = Router();
const userController = new UserController();

router.post("/register", (req, res) => userController.registerUser(req, res));

export default router;

// src/routes/payment.routes.ts
import { Router } from "express";
import { PaymentController } from "../controllers/payment.controller";

const router = Router();
const paymentController = new PaymentController();

router.post("/pay", (req, res) => paymentController.handlePayment(req, res));

export default router;


---

8. Server Setup

// src/server.ts
import express from "express";
import { AppDataSource } from "./database/connection";
import userRoutes from "./routes/user.routes";
import paymentRoutes from "./routes/payment.routes";

const app = express();
app.use(express.json());

// Routes
app.use("/api/users", userRoutes);
app.use("/api/payments", paymentRoutes);

// Start Server
AppDataSource.initialize()
    .then(() => {
        console.log("Database connected");
        app.listen(5000, () => console.log("Server running on port 5000"));
    })
    .catch((error) => console.log(error));


---

Frontend (React + TypeScript)

1. Registration Form and Payment Component

// src/components/App.tsx
import React, { useState } from "react";
import axios from "axios";

const App: React.FC = () => {
    const [formData, setFormData] = useState({
        name: "",
        email: "",
        username: "",
        password: "",
        pinCode: "",
        country: "",
        address: "",
    });
    const [paymentDetails, setPaymentDetails] = useState({ priceId: "", quantity: 1 });

    const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        setFormData({ ...formData, [e.target.name]: e.target.value });
    };

    const handlePaymentChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        setPaymentDetails({ ...paymentDetails, [e.target.name]: e.target.value });
    };

    const handleSubmit = async () => {
        try {
            const { data: paymentResponse } = await axios.post("/api/payments/pay", paymentDetails);
            if (paymentResponse.success) {
                await axios.post("/api/users/register", formData);
                alert("User created and payment successful!");
            }
        } catch (error) {
            alert("Error occurred: " + error.message);
        }
    };

    return (
        <div style={{ display: "flex" }}>
            {/* Registration Form */}
            <div>
                <h2>Register</h2>
                {Object.keys(formData).map((key) => (
                    <input
                        key={key}
                        name={key}
                        placeholder={key}
                        value={formData[key]}
                        onChange={handleInputChange}
                    />
                ))}
            </div>
            {/* Payment Details */}
            <div>
                <h2>Payment</h2>
                <input
                    name="priceId"
                    placeholder="Price ID"
                    value={paymentDetails.priceId}
                    onChange={handlePaymentChange}
                />
                <input
                    name="quantity"
                    type="number"
                    placeholder="Quantity"
                    value={paymentDetails.quantity}
                    onChange={handlePaymentChange}
                />
                <button onClick={handleSubmit}>Pay Now</button>
            </div>
        </div>
    );
};

export default App;


---

This is a high-level implementation of your project. Customize and enhance it as needed for your use case!

Here's how to structure the frontend to have two components (Signup and Payment), and use them together in a single Checkout page. The Payment component will receive the email and name entered in the Signup component.


---

Folder Structure

src/
├── components/
│   ├── Signup.tsx
│   ├── Payment.tsx
│   ├── Checkout.tsx
├── App.tsx


---

1. Signup Component

The Signup component collects user information and passes it to the Checkout component.

// src/components/Signup.tsx
import React, { useState } from "react";

interface SignupProps {
    onSignupComplete: (data: { name: string; email: string }) => void;
}

const Signup: React.FC<SignupProps> = ({ onSignupComplete }) => {
    const [formData, setFormData] = useState({
        name: "",
        email: "",
        username: "",
        password: "",
        pinCode: "",
        country: "",
        address: "",
    });

    const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        setFormData({ ...formData, [e.target.name]: e.target.value });
    };

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        // Pass name and email to the parent
        onSignupComplete({ name: formData.name, email: formData.email });
    };

    return (
        <div>
            <h2>Signup</h2>
            <form onSubmit={handleSubmit}>
                {Object.keys(formData).map((key) => (
                    <div key={key}>
                        <label>{key}</label>
                        <input
                            name={key}
                            placeholder={key}
                            value={formData[key]}
                            onChange={handleInputChange}
                            required={key === "name" || key === "email" || key === "password"} // Add validation for required fields
                        />
                    </div>
                ))}
                <button type="submit">Continue</button>
            </form>
        </div>
    );
};

export default Signup;


---

2. Payment Component

The Payment component accepts name and email from the Signup component as props.

// src/components/Payment.tsx
import React, { useState } from "react";
import axios from "axios";

interface PaymentProps {
    name: string;
    email: string;
}

const Payment: React.FC<PaymentProps> = ({ name, email }) => {
    const [paymentDetails, setPaymentDetails] = useState({
        priceId: "",
        quantity: 1,
    });

    const handlePaymentChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        setPaymentDetails({ ...paymentDetails, [e.target.name]: e.target.value });
    };

    const handlePayment = async () => {
        try {
            const paymentResponse = await axios.post("/api/payments/pay", {
                ...paymentDetails,
                customerDetails: { name, email },
            });

            if (paymentResponse.data.success) {
                alert("Payment successful!");
            }
        } catch (error) {
            alert("Payment failed: " + error.message);
        }
    };

    return (
        <div>
            <h2>Payment</h2>
            <div>
                <label>Price ID</label>
                <input
                    name="priceId"
                    placeholder="Price ID"
                    value={paymentDetails.priceId}
                    onChange={handlePaymentChange}
                />
            </div>
            <div>
                <label>Quantity</label>
                <input
                    name="quantity"
                    type="number"
                    placeholder="Quantity"
                    value={paymentDetails.quantity}
                    onChange={handlePaymentChange}
                />
            </div>
            <button onClick={handlePayment}>Pay Now</button>
        </div>
    );
};

export default Payment;


---

3. Checkout Page

The Checkout page brings both components together. It uses state to pass the name and email from Signup to Payment.

// src/components/Checkout.tsx
import React, { useState } from "react";
import Signup from "./Signup";
import Payment from "./Payment";

const Checkout: React.FC = () => {
    const [userDetails, setUserDetails] = useState<{ name: string; email: string } | null>(null);

    const handleSignupComplete = (data: { name: string; email: string }) => {
        setUserDetails(data);
    };

    return (
        <div style={{ display: "flex", justifyContent: "space-between" }}>
            <div style={{ width: "45%" }}>
                <Signup onSignupComplete={handleSignupComplete} />
            </div>
            <div style={{ width: "45%" }}>
                {userDetails ? (
                    <Payment name={userDetails.name} email={userDetails.email} />
                ) : (
                    <p>Please complete the signup to proceed to payment.</p>
                )}
            </div>
        </div>
    );
};

export default Checkout;


---

4. App Component

Import the Checkout component into your App component.

// src/App.tsx
import React from "react";
import Checkout from "./components/Checkout";

const App: React.FC = () => {
    return (
        <div>
            <h1>Checkout Page</h1>
            <Checkout />
        </div>
    );
};

export default App;


---

Explanation

1. Signup Component:

Collects user data.

Passes the name and email to the parent (Checkout) when the form is submitted.



2. Payment Component:

Receives name and email as props from the Checkout component.

Uses axios to send payment details along with the user's name and email to the backend.



3. Checkout Component:

Combines Signup and Payment.

Maintains the name and email in its state and passes them from Signup to Payment.



4. App Component:

Serves as the root component and renders the Checkout page.





---

This setup ensures a clean separation of concerns while maintaining a clear data flow between components. Let me know if you'd like further assistance!

