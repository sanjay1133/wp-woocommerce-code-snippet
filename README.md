<div align="center"><h1><a href="https://www.galaxyweblinks.com/" target="_blank">Galaxy Weblinks</a></h1></div>

# WordPress - Snippets

![Licence](https://img.shields.io/badge/Unlicense-red)

## Overview

This is a list of useful **WooCommerce** code snippets and functions that I often reference to enhance or clean up my sites. 

**Note:** Please be careful and make backups!


## WooCommerce

- [Create A Message For Remaining Amount Of A Purchase For Free Delivery In WooCommerce](#create-a-message-for-remaining-amount-of-a-purchase-for-free-delivery-in-woocommerce)
- [Change The Appearance Of A Foreign Currency In WooCommerce](#change-the-appearance-of-a-foreign-currency-in-woocommerce)
- [Remove Specific Product Tabs In WooCommerce](#remove-specific-product-tabs-in-woocommerce)
- [Add A Message To The Login Or Registration Form In WooCommerce](#add-a-message-to-the-login-or-registration-form-in-woocommerce)
- [Display All Products Purchased By User Via Shortcode In WooCommerce](#display-all-products-purchased-by-user-via-shortcode-in-woocommerce)
- [How To Add Custom Post Type To WooCommerce](#how-to-add-custom-post-type-to-woocommerce)
- [How To Add A New Tab At My Account Page In WooCommerce](#how-to-add-a-new-tab-at-my-account-page-in-woocommerce)
- [How To Reorder A Custom Tab At My Account Page In WooCommerce](#how-to-reorder-a-custom-tab-at-my-account-page-in-woocommerce)

---

## WooCommerce

### Change The Appearance Of A Foreign Currency In WooCommerce

```php
function wc_change_bgn_currency_symbol( $currency_symbol, $currency ) {
  switch ( $currency ) {
    case 'BGN':
      $currency_symbol = 'BGN';
    break;
  }
  
  return $currency_symbol;
}

add_filter( 'woocommerce_currency_symbol', 'wc_change_bgn_currency_symbol', 10, 2 );
```

### Create A Message For Remaining Amount Of A Purchase For Free Delivery In WooCommerce

```php
/**
 * Notice with $$$ remaining to Free Shipping @ WooCommerce Cart 
 * Tested with WooCommerce version 3.0.5
 */ 
function wc_free_shipping_cart_notice() { 
  global $woocommerce; 
  // Get Free Shipping Methods for Rest of the World Zone & populate array $min_amounts
  $default_zone = new WC_Shipping_Zone(0); 
  $default_methods = $default_zone->get_shipping_methods();
    foreach( $default_methods as $key => $value ) {
      if ( $value->id === "free_shipping" ) {
        if ( $value->min_amount > 0 ) $min_amounts[] = $value->min_amount;
      }
    }
    
    // Get Free Shipping Methods for all other ZONES & populate array $min_amounts
    $delivery_zones = WC_Shipping_Zones::get_zones();
    foreach ( $delivery_zones as $key => $delivery_zone ) {
      foreach ( $delivery_zone['shipping_methods'] as $key => $value ) {
        if ( $value->id === "free_shipping" ) {
          if ( $value->min_amount > 0 ) $min_amounts[] = $value->min_amount;
        }
      }
    }
    
    // Find lowest min_amount
    if ( is_array($min_amounts) ) {
       $min_amount = min($min_amounts);
       // Get Cart Subtotal inc. Tax excl. Shipping
       $current = WC()->cart->subtotal;
       // If Subtotal < Min Amount, еcho Notice and add "Continue Shopping" button
       if ( $current < $min_amount ) {
          $added_text = esc_html__('You need to add ', 'woocommerce' ) . wc_price( $min_amount - $current ) . esc_html__(' for free delivery!', 'woocommerce' );
    $return_to = apply_filters( 'woocommerce_continue_shopping_redirect', wc_get_raw_referer() ? wp_validate_redirect( wc_get_raw_referer(), false ) : wc_get_page_permalink( '/' ) );
    $notice = sprintf( '%s %s', esc_url( $return_to ), esc_html__( 'Continue shopping', 'woocommerce' ), $added_text );
    wc_print_notice( $notice, 'notice' );
       }
    }
}

add_action( 'woocommerce_before_cart', 'wc_free_shipping_cart_notice' );
```

### Remove Specific Product Tabs In WooCommerce

```php
function wc_remove_product_tabs( $tabs ) {
   // remove the description tab
   unset( $tabs['description'] );
   
   // remove the reviews tab
   unset( $tabs['reviews'] );
   
   // remove the additional information tab
   unset( $tabs['additional_information'] );
   
   return $tabs;
}

add_filter( 'woocommerce_product_tabs', 'wc_remove_product_tabs', 99 );
```

### Add A Message To The Login Or Registration Form In WooCommerce

```php
function wc_custom_login_message() {
   if ( get_option( 'woocommerce_enable_myaccount_registration' ) == 'yes' ) {
       $html  = '<div class="woocommerce-info">';
       $html .= '<p>' . _e( 'Your custom message goes here.' ) . '</p>';
       $html .= '</div>';
       echo $html;
   }
}

add_action( 'woocommerce_before_customer_login_form', 'wc_custom_login_message' );
```

### Display All Products Purchased By User Via Shortcode In WooCommerce

```php 
add_shortcode( 'my_products', 'wc_user_products_bought' );
  
function wc_user_products_bought() {
  
    global $product, $woocommerce, $woocommerce_loop;
    $columns = 3;
  
    // Get user
    $current_user = wp_get_current_user();
  
    // Get user orders (COMPLETED + PROCESSING)
    $customer_orders = get_posts( array(
        'numberposts' => -1,
        'meta_key'    => '_customer_user',
        'meta_value'  => $current_user->ID,
        'post_type'   => wc_get_order_types(),
        'post_status' => array_keys( wc_get_is_paid_statuses() ),
    ) );
  
    // Loop Through orders and get product IDs
    if ( ! $customer_orders ) return;
    
    $product_ids = array();
    
    foreach ( $customer_orders as $customer_order ) {
        $order = wc_get_order( $customer_order->ID );
        $items = $order->get_items();
        foreach ( $items as $item ) {
            $product_id = $item->get_product_id();
            $product_ids[] = $product_id;
        }
    }
    $product_ids = array_unique( $product_ids );
  
    // Query products
    $args = array(
       'post_type' => 'product',
       'post__in' => $product_ids,
    );
    
    $loop = new WP_Query( $args );
  
    // Generate WC loop
    ob_start();
    woocommerce_product_loop_start();
    while ( $loop->have_posts() ) : $loop->the_post();
    wc_get_template_part( 'content', 'product' ); 
    endwhile; 
    woocommerce_product_loop_end();
    woocommerce_reset_loop();
    wp_reset_postdata();
  
    // Return content
    return '<div class="woocommerce columns-' . $columns . '">' . ob_get_clean() . '</div>';
}
```

### How To Add Custom Post Type To WooCommerce

**Step 1**

```php
/**
 * Your post type should have a custom field called 'price'. 
 * We just have to make sure that its meta key is '_price'. 
 * How to check that? You can try to inspect the element:  
 * 1) Visit your post type admin page. 
 * 2) Try to add or edit a post in your post type admin page. 
 * 3) Look for the text box that has the price label and right click on the text box. 
 * 4) You should find it's name attribute to be '_price'. 
 * Usually the name is the meta key for a custom field.
 *
 * Add the code below if your meta key is not '_price'
 */
add_filter('woocommerce_get_price','wc_get_price', 20, 2);

function wc_get_price($price, $post) {
  if ($post->post->post_type === 'post') {
    $price = get_post_meta($post->id, 'price', true);
  }
  
  return $price;
}
```

**Step 2**

```php
/**
 * You're now able to add a custom post type to the cart.
 * Now we have to create our own "Add to Cart" button.
 */
add_filter('the_content', 'wc_add_to_cart_button', 20, 1);

function wc_add_to_cart_button($content) {
  global $post;
 
  if ($post->post_type !== 'post') {
    return $content;
  }

  ob_start(); ?>
 
  <form action="" method="post">
    <input name="add-to-cart" type="hidden" value="<?php echo $post->ID ?>">
    <input name="quantity" type="number" value="1" min="1">
    <input name="submit" type="submit" value="Add to cart>
  </form><?php
	
  return $content . ob_get_clean();
}
```

**Note:** You have to call `wc_print_notices()` function in your `single-{post_type}.php`. This will display the messages like: **"'Hello world!' has been added to your cart."**.

### How To Add A New Tab At My Account Page In WooCommerce

```php
/**
 * Register new endpoint slug to use for My Account page
 */

/**
 * @important-note	Resave permalinks or it will give 404 error
 */
function woo_custom_add_premium_support_endpoint() {
   add_rewrite_endpoint( 'premium-support', EP_ROOT | EP_PAGES );
}
  
add_action( 'init', 'woo_custom_add_premium_support_endpoint' );
  
/**
 * Add new query var
 */
  
function woo_custom_premium_support_query_vars( $vars ) {
   $vars[] = 'premium-support';
   return $vars;
}
  
add_filter( 'woocommerce_get_query_vars', 'woo_custom_premium_support_query_vars', 0 );
  
/**
 * Insert the new endpoint into the My Account menu
 */
function woo_custom_add_premium_support_link_my_account( $items ) {
   $items['premium-support'] = 'Premium Support';
   return $items;
}
  
add_filter( 'woocommerce_account_menu_items', 'woo_custom_add_premium_support_link_my_account' );
  
/**
 * Add content to the new endpoint
 */
function woo_custom_premium_support_content() {
   echo '<h3>Premium WooCommerce Support</h3><p>Welcome to the WooCommerce support area.</p>';
   echo do_shortcode( ' /* your shortcode here */ ' );
}

/**
 * @important-note	"add_action" must follow 'woocommerce_account_{your-endpoint-slug}_endpoint' format
 */
add_action( 'woocommerce_account_premium-support_endpoint', 'woo_custom_premium_support_content' );
```

### How To Reorder A Custom Tab At My Account Page In WooCommerce

```php
/**
 * Rename or re-order my account menu items
 */
function woo_reorder_my_account_menu() {
    $neworder = array(
        'dashboard'          => __( 'Dashboard', 'woocommerce' ),
        'orders'             => __( 'Previous Orders', 'woocommerce' ),
        'custom-tab'         => __( 'Custom Tab', 'woocommerce' ),
        'edit-address'       => __( 'Addresses', 'woocommerce' ),
        'edit-account'       => __( 'Account Details', 'woocommerce' ),
        'customer-logout'    => __( 'Logout', 'woocommerce' ),
    );
    
    return $neworder;
}

add_filter ( 'woocommerce_account_menu_items', 'woo_reorder_my_account_menu' );
```

---

## License

This repository is unlicense[d], so feel free to fork.
