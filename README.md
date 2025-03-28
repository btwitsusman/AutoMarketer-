# AutoMarketer-

require("dotenv").config(); // Load environment variables
const express = require("express");
const cors = require("cors");
const axios = require("axios");
const rateLimit = require("express-rate-limit");
const { body, validationResult } = require("express-validator");

const app = express();
const PORT = process.env.PORT || 5000;

// Middleware
app.use(cors());
app.use(express.json());

// Rate Limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // Limit each IP to 100 requests per windowMs
    message: { success: false, error: "Too many requests, please try again later." }
});
app.use(limiter);

// WhatsApp API Integration (Twilio Example)
app.post(
    "/send-whatsapp",
    [
        body("to").notEmpty().withMessage("Recipient number is required."),
        body("message").notEmpty().withMessage("Message is required.")
    ],
    async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ success: false, errors: errors.array() });
        }

        const { to, message } = req.body;
        try {
            const response = await axios.post(
                `https://api.twilio.com/2010-04-01/Accounts/${process.env.TWILIO_ACCOUNT_SID}/Messages.json`,
                {
                    To: `whatsapp:${to}`,
                    From: `whatsapp:${process.env.TWILIO_NUMBER}`,
                    Body: message
                },
                {
                    auth: {
                        username: process.env.TWILIO_ACCOUNT_SID,
                        password: process.env.TWILIO_AUTH_TOKEN
                    }
                }
            );
            res.json({ success: true, response: response.data });
        } catch (error) {
            res.status(500).json({ success: false, error: error.response?.data || error.message });
        }
    }
);

// Facebook Post Automation (Meta API)
app.post(
    "/post-facebook",
    [
        body("pageId").notEmpty().withMessage("Page ID is required."),
        body("message").notEmpty().withMessage("Message is required."),
        body("accessToken").notEmpty().withMessage("Access token is required.")
    ],
    async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ success: false, errors: errors.array() });
        }

        const { pageId, message, accessToken } = req.body;
        try {
            const response = await axios.post(`https://graph.facebook.com/${pageId}/feed`, {
                message,
                access_token: accessToken
            });
            res.json({ success: true, response: response.data });
        } catch (error) {
            res.status(500).json({ success: false, error: error.response?.data || error.message });
        }
    }
);

// Stripe Payment Integration
app.post(
    "/create-payment",
    [
        body("amount").isNumeric().withMessage("Amount must be a number."),
        body("currency").notEmpty().withMessage("Currency is required.")
    ],
    async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ success: false, errors: errors.array() });
        }

        const { amount, currency } = req.body;
        try {
            const response = await axios.post(
                "https://api.stripe.com/v1/payment_intents",
                new URLSearchParams({ amount, currency }).toString(),
                {
                    headers: {
                        Authorization: `Bearer ${process.env.STRIPE_SECRET_KEY}`,
                        "Content-Type": "application/x-www-form-urlencoded"
                    }
                }
            );
            res.json({ success: true, clientSecret: response.data.client_secret });
        } catch (error) {
            res.status(500).json({ success: false, error: error.response?.data || error.message });
        }
    }
);

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
