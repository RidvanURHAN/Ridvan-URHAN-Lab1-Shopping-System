START E_Commerce_Checkout_System

    // KONTROL NOKTASI 1: Kullanıcı Girişi ve Oturum Durumu
    START User_Login
        IF user_credentials_are_valid THEN
            Generate_Session_Token()
            session_active = TRUE
        ELSE
            Print "Giriş Başarısız. Lütfen bilgilerinizi kontrol edin."
            Exit System
        END IF
    END User_Login

    // KONTROL NOKTASI 2: Stok Yeterliliği ve Sepete Ekleme
    START Add_To_Cart
        LOOP while user_wants_to_add_products
            IF product_stock >= requested_quantity THEN
                Insert_Item_To_Cart(product_id, requested_quantity)
                Update_Cart_Subtotal()
            ELSE
                Print "Hata: İstenen miktarda stok bulunmamaktadır."
            END IF
        END LOOP
    END Add_To_Cart

    // KONTROL NOKTASI 3: İndirim Kodu Geçerliliği ve Şartları
    START Apply_Discount
        IF user_enters_discount_code THEN
            IF code_is_valid AND code_not_expired AND cart_meets_conditions THEN
                cart_subtotal = cart_subtotal - discount_amount
                Print "İndirim başarıyla uygulandı."
            ELSE
                Print "Geçersiz, süresi dolmuş veya bu sepet için uygun olmayan kod."
            END IF
        END IF
    END Apply_Discount

    // KONTROL NOKTASI 4: Kargo Barajı ve Maliyet Hesaplama
    START Calculate_Shipping
        shipping_fee = Calculate_Base_Rate(weight, delivery_address)
        
        IF cart_subtotal >= free_shipping_threshold THEN
            shipping_fee = 0
            Print "Kargo Bedava!"
        END IF
        
        final_total = cart_subtotal + shipping_fee
    END Calculate_Shipping

    // KONTROL NOKTASI 5: Ödeme Güvenliği ve Veri Bütünlüğü (Transaction)
    START Process_Payment
        // Ödeme öncesi eşzamanlılık (race condition) için son stok kontrolü
        LOOP for each item in Cart
            IF current_database_stock < item.quantity THEN
                Print "Hata: Ödeme öncesi bir ürünün stoğu tükendi."
                Exit System
            END IF
        END LOOP

        payment_status = Request_Bank_Gateway(final_total, credit_card_info)

        IF payment_status == SUCCESS THEN
            // Veritabanı işlemleri (Atomicity)
            Deduct_Stock_From_Inventory(Cart)
            Save_Order_Records()
            Clear_User_Cart()
            Print "Ödeme alındı. Siparişiniz başarıyla oluşturuldu."
        ELSE
            Print "Ödeme reddedildi. Lütfen kart limitinizi kontrol edin."
        END IF
    END Process_Payment

END E_Commerce_Checkout_System
