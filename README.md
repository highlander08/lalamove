

```php

### versao com origem sao paulo e apenas 1 vendedor e 1 cliente

function testar_lalamove_manual() {
 if (isset($_GET['test_lalamove']) && current_user_can('manage_options')) {
     echo '<div style="padding: 20px; background: #f0f0f1; margin: 20px; border-radius: 5px;">';
     echo '<h1>🚀 Teste Lalamove no WordPress</h1>';
     
     $api_key = 'pk_test_405b82b18afd1a3b961cb61c82e063cf';
     $secret = 'sk_test_8exxWZ9sH70NtO5Kt3yHsp9aDVY5OkhG0/J2hRDuxpXKkatG6OiwEZvA5Wr5TcGF';
     $market = 'BR';
     $base_url = 'https://rest.sandbox.lalamove.com';

     // Função para gerar UUID
     function generateUUID() {
         return sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
             mt_rand(0, 0xffff), mt_rand(0, 0xffff),
             mt_rand(0, 0xffff), mt_rand(0, 0x0fff) | 0x4000,
             mt_rand(0, 0x3fff) | 0x8000, mt_rand(0, 0xffff), 
             mt_rand(0, 0xffff), mt_rand(0, 0xffff)
         );
     }

     // Função para fazer requests
     function makeLalamoveRequest($url, $headers, $body = null, $method = 'POST') {
         $args = [
             'headers' => $headers,
             'timeout' => 30
         ];
         
         if ($body && $method === 'POST') {
             $args['body'] = $body;
         }
         
         if ($method === 'POST') {
             $response = wp_remote_post($url, $args);
         } else {
             $response = wp_remote_get($url, $args);
         }
         
         if (is_wp_error($response)) {
             throw new Exception('Erro WordPress: ' . $response->get_error_message());
         }
         
         return [
             'status' => wp_remote_retrieve_response_code($response),
             'body' => wp_remote_retrieve_body($response)
         ];
     }

     try {
         // 1️⃣ Criar Cotação
         echo '<h3>📦 Criando Cotação...</h3>';
         
         $quotationBody = [
             "data" => [
                 "serviceType" => "CAR",
                 "specialRequests" => [],
                 "language" => "pt_BR",
                 "stops" => [
                     [
                         "coordinates" => ["lat" => "-23.5505", "lng" => "-46.6333"],
                         "address" => "Av. Paulista, 1000 - São Paulo"
                     ],
                     [
                         "coordinates" => ["lat" => "-23.5510", "lng" => "-46.6340"],
                         "address" => "Rua Augusta, 500 - São Paulo"
                     ]
                 ],
                 "isRouteOptimized" => false
             ]
         ];

         $bodyString = json_encode($quotationBody);
         $time = round(microtime(true) * 1000);
         $signature = hash_hmac('sha256', $time . "\r\nPOST\r\n/v3/quotations\r\n\r\n" . $bodyString, $secret);
         $token = $api_key . ':' . $time . ':' . $signature;

         $headers = [
             'Authorization' => 'hmac ' . $token,
             'Market' => $market,
             'Request-ID' => generateUUID(),
             'Content-Type' => 'application/json'
         ];

         $quotationResult = makeLalamoveRequest($base_url . '/v3/quotations', $headers, $bodyString, 'POST');

         if ($quotationResult['status'] !== 201) {
             throw new Exception("Erro na cotação: " . $quotationResult['status'] . " - " . $quotationResult['body']);
         }

         $quotationData = json_decode($quotationResult['body'], true);
         
         echo '<div style="background: #d4edda; padding: 10px; border-radius: 5px;">';
         echo '<p><strong>✅ COTAÇÃO CRIADA COM SUCESSO!</strong></p>';
         echo '<p><strong>ID:</strong> ' . $quotationData['data']['quotationId'] . '</p>';
         echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' ' . $quotationData['data']['priceBreakdown']['currency'] . '</p>';
         echo '</div>';

         // 2️⃣ Criar Ordem
         echo '<h3>🎯 Criando Ordem...</h3>';
         
         $orderBody = [
             "data" => [
                 "quotationId" => $quotationData['data']['quotationId'],
                 "sender" => [
                     "stopId" => $quotationData['data']['stops'][0]['stopId'],
                     "name" => "Loja WordPress",
                     "phone" => "+5511999999999"
                 ],
                 "recipients" => [
                     [
                         "stopId" => $quotationData['data']['stops'][1]['stopId'],
                         "name" => "Cliente WordPress",
                         "phone" => "+5511888888888",
                         "remarks" => "Entregar na portaria"
                     ]
                 ],
                 "isPODEnabled" => false,
                 "metadata" => [
                     "orderReference" => "WP-TEST-" . time(),
                     "customerName" => "Teste WordPress"
                 ]
             ]
         ];

         $orderBodyString = json_encode($orderBody);
         $time2 = round(microtime(true) * 1000);
         $signature2 = hash_hmac('sha256', $time2 . "\r\nPOST\r\n/v3/orders\r\n\r\n" . $orderBodyString, $secret);
         $token2 = $api_key . ':' . $time2 . ':' . $signature2;

         $headers2 = [
             'Authorization' => 'hmac ' . $token2,
             'Market' => $market,
             'Request-ID' => generateUUID(),
             'Content-Type' => 'application/json'
         ];

         $orderResult = makeLalamoveRequest($base_url . '/v3/orders', $headers2, $orderBodyString, 'POST');

         if ($orderResult['status'] !== 201) {
             throw new Exception("Erro na ordem: " . $orderResult['status'] . " - " . $orderResult['body']);
         }

         $orderData = json_decode($orderResult['body'], true);
         
         echo '<div style="background: #d1ecf1; padding: 10px; border-radius: 5px;">';
         echo '<p><strong>✅ ORDEM CRIADA COM SUCESSO!</strong></p>';
         echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
         echo '<p><strong>Status:</strong> ' . $orderData['data']['status'] . '</p>';
         if (isset($orderData['data']['shareLink'])) {
             echo '<p><a href="' . $orderData['data']['shareLink'] . '" target="_blank">Acompanhar Entrega</a></p>';
         }
         echo '</div>';

         // 3️⃣ Consultar Pedido
         echo '<h3>🔍 Verificando Pedido no Sandbox...</h3>';
         $orderId = $orderData['data']['orderId'];

         $time3 = round(microtime(true) * 1000);
         $path = '/v3/orders/' . $orderId;
         $signature3 = hash_hmac('sha256', $time3 . "\r\nGET\r\n" . $path . "\r\n\r\n", $secret);
         $token3 = $api_key . ':' . $time3 . ':' . $signature3;

         $headers3 = [
             'Authorization' => 'hmac ' . $token3,
             'Market' => $market,
             'Request-ID' => generateUUID(),
             'Content-Type' => 'application/json'
         ];

         $check = makeLalamoveRequest($base_url . $path, $headers3, null, 'GET');
         echo '<pre style="background:#fff; padding:10px; border-radius:5px;">Consulta: ' . htmlentities($check['body']) . '</pre>';

         echo '<div style="background: #fff3cd; padding: 15px; border-radius: 5px;">';
         echo '<h3>🎉 RESUMO FINAL</h3>';
         echo '<p><strong>Quotation ID:</strong> ' . $quotationData['data']['quotationId'] . '</p>';
         echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
         echo '<p><strong>Status:</strong> ' . $orderData['data']['status'] . '</p>';
         echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' ' . $quotationData['data']['priceBreakdown']['currency'] . '</p>';
         echo '</div>';

     } catch (Exception $e) {
         echo '<div style="background: #f8d7da; padding: 10px; border-radius: 5px;">';
         echo '<p><strong>❌ ERRO:</strong> ' . $e->getMessage() . '</p>';
         echo '</div>';
     }

     echo '<hr>';
     echo '<p><strong>💡 Para testar novamente:</strong> <a href="' . admin_url('?test_lalamove=1') . '">Clique aqui</a></p>';
     echo '</div>';
     die();
 }
}
add_action('init', 'testar_lalamove_manual');


### versao com origem fortaleza 1 vendedor e 1 cliente

  function testar_lalamove_manual() {
    if (isset($_GET['test_lalamove']) && current_user_can('manage_options')) {
        
        echo '<div style="padding: 20px; background: #f0f0f1; margin: 20px; border-radius: 5px; font-family: Arial, sans-serif;">';
        echo '<h1>🚀 Teste Lalamove - Fortaleza (CE)</h1>';
        
        $api_key = 'pk_test_405b82b18afd1a3b961cb61c82e063cf';
        $secret = 'sk_test_8exxWZ9sH70NtO5Kt3yHsp9aDVY5OkhG0/J2hRDuxpXKkatG6OiwEZvA5Wr5TcGF';
        $market = 'BR';
        $base_url = 'https://rest.sandbox.lalamove.com';

        // Função para gerar UUID
        function generateUUID() {
            return sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
                mt_rand(0, 0xffff), mt_rand(0, 0xffff),
                mt_rand(0, 0xffff), mt_rand(0, 0x0fff) | 0x4000,
                mt_rand(0, 0x3fff) | 0x8000, mt_rand(0, 0xffff), 
                mt_rand(0, 0xffff), mt_rand(0, 0xffff)
            );
        }

        // Função para fazer requests
        function makeLalamoveRequest($url, $headers, $body = null, $method = 'POST') {
            $args = [
                'headers' => $headers,
                'timeout' => 30
            ];
            
            if ($body && $method === 'POST') {
                $args['body'] = $body;
            }
            
            if ($method === 'POST') {
                $response = wp_remote_post($url, $args);
            } else {
                $response = wp_remote_get($url, $args);
            }
            
            if (is_wp_error($response)) {
                throw new Exception('Erro WordPress: ' . $response->get_error_message());
            }
            
            return [
                'status' => wp_remote_retrieve_response_code($response),
                'body' => wp_remote_retrieve_body($response)
            ];
        }

        try {
            // 1. CRIAR COTAÇÃO
            echo '<h3>📦 Criando Cotação (Fortaleza)...</h3>';
            
            $quotationBody = [
                "data" => [
                    "serviceType" => "CAR",
                    "specialRequests" => [],
                    "language" => "pt_BR",
                    "stops" => [
                        [
                            "coordinates" => [
                                "lat" => "-3.7319",  // Praia de Iracema
                                "lng" => "-38.5267"
                            ],
                            "address" => "Av. Beira Mar, 3000 - Meireles, Fortaleza - CE"
                        ],
                        [
                            "coordinates" => [
                                "lat" => "-3.7921",  // Shopping Iguatemi
                                "lng" => "-38.4803"
                            ],
                            "address" => "Av. Washington Soares, 85 - Edson Queiroz, Fortaleza - CE"
                        ]
                    ],
                    "isRouteOptimized" => false
                ]
            ];

            $bodyString = json_encode($quotationBody);
            $time = round(microtime(true) * 1000);
            $signature = hash_hmac('sha256', $time . "\r\nPOST\r\n/v3/quotations\r\n\r\n" . $bodyString, $secret);
            $token = $api_key . ':' . $time . ':' . $signature;

            $headers = [
                'Authorization' => 'hmac ' . $token,
                'Market' => $market,
                'Request-ID' => generateUUID(),
                'Content-Type' => 'application/json'
            ];

            $quotationResult = makeLalamoveRequest($base_url . '/v3/quotations', $headers, $bodyString, 'POST');

            if ($quotationResult['status'] !== 201) {
                throw new Exception("Erro na cotação: " . $quotationResult['status'] . " - " . $quotationResult['body']);
            }

            $quotationData = json_decode($quotationResult['body'], true);
            
            echo '<div style="background: #d4edda; padding: 10px; border-radius: 5px; margin: 10px 0;">';
            echo '<p><strong>✅ COTAÇÃO CRIADA COM SUCESSO!</strong></p>';
            echo '<p><strong>ID:</strong> ' . $quotationData['data']['quotationId'] . '</p>';
            echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' ' . $quotationData['data']['priceBreakdown']['currency'] . '</p>';
            echo '</div>';

            // 2. CRIAR ORDEM
            echo '<h3>🎯 Criando Ordem...</h3>';
            
            $orderBody = [
                "data" => [
                    "quotationId" => $quotationData['data']['quotationId'],
                    "sender" => [
                        "stopId" => $quotationData['data']['stops'][0]['stopId'],
                        "name" => "Loja Fortaleza",
                        "phone" => "+5585999999999"
                    ],
                    "recipients" => [
                        [
                            "stopId" => $quotationData['data']['stops'][1]['stopId'],
                            "name" => "Cliente Fortaleza",
                            "phone" => "+5585888888888",
                            "remarks" => "Entregar na recepção do shopping"
                        ]
                    ],
                    "isPODEnabled" => false,
                    "metadata" => [
                        "orderReference" => "FORT-TEST-" . time(),
                        "customerName" => "Teste Fortaleza"
                    ]
                ]
            ];

            $orderBodyString = json_encode($orderBody);
            $time2 = round(microtime(true) * 1000);
            $signature2 = hash_hmac('sha256', $time2 . "\r\nPOST\r\n/v3/orders\r\n\r\n" . $orderBodyString, $secret);
            $token2 = $api_key . ':' . $time2 . ':' . $signature2;

            $headers2 = [
                'Authorization' => 'hmac ' . $token2,
                'Market' => $market,
                'Request-ID' => generateUUID(),
                'Content-Type' => 'application/json'
            ];

            $orderResult = makeLalamoveRequest($base_url . '/v3/orders', $headers2, $orderBodyString, 'POST');

            if ($orderResult['status'] !== 201) {
                throw new Exception("Erro na ordem: " . $orderResult['status'] . " - " . $orderResult['body']);
            }

            $orderData = json_decode($orderResult['body'], true);
            
            echo '<div style="background: #d1ecf1; padding: 10px; border-radius: 5px; margin: 10px 0;">';
            echo '<p><strong>✅ ORDEM CRIADA COM SUCESSO!</strong></p>';
            echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
            echo '<p><strong>Status:</strong> ' . $orderData['data']['status'] . '</p>';
            if (isset($orderData['data']['shareLink'])) {
                echo '<p><strong>Link:</strong> <a href="' . $orderData['data']['shareLink'] . '" target="_blank">Acompanhar Entrega</a></p>';
            }
            echo '</div>';

            // 3. CONSULTAR PEDIDO
            echo '<h3>🔍 Verificando Pedido no Sandbox...</h3>';
            $orderId = $orderData['data']['orderId'];

            $time3 = round(microtime(true) * 1000);
            $path = '/v3/orders/' . $orderId;
            $signature3 = hash_hmac('sha256', $time3 . "\r\nGET\r\n" . $path . "\r\n\r\n", $secret);
            $token3 = $api_key . ':' . $time3 . ':' . $signature3;

            $headers3 = [
                'Authorization' => 'hmac ' . $token3,
                'Market' => $market,
                'Request-ID' => generateUUID(),
                'Content-Type' => 'application/json'
            ];

            $check = makeLalamoveRequest($base_url . $path, $headers3, null, 'GET');
            echo '<pre style="background:#fff; padding:10px; border-radius:5px; overflow-x:auto;">Consulta: ' . htmlentities($check['body']) . '</pre>';

            // RESUMO FINAL
            echo '<div style="background: #fff3cd; padding: 15px; border-radius: 5px; margin: 20px 0;">';
            echo '<h3>🎉 RESUMO FINAL (Fortaleza)</h3>';
            echo '<p><strong>Quotation ID:</strong> ' . $quotationData['data']['quotationId'] . '</p>';
            echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
            echo '<p><strong>Status:</strong> ' . $orderData['data']['status'] . '</p>';
            echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' ' . $quotationData['data']['priceBreakdown']['currency'] . '</p>';
            echo '</div>';

        } catch (Exception $e) {
            echo '<div style="background: #f8d7da; padding: 10px; border-radius: 5px; margin: 10px 0;">';
            echo '<p><strong>❌ ERRO:</strong> ' . htmlspecialchars($e->getMessage()) . '</p>';
            echo '</div>';
        }
        
        echo '<hr>';
        echo '<p><strong>💡 Testar novamente:</strong> <a href="' . admin_url('?test_lalamove=1') . '">Clique aqui</a></p>';
        echo '</div>';
        
        die();
    }
}
add_action('init', 'testar_lalamove_manual');


### versao fortaleza com 2 vendedores e 1 cliente

 // === MULTI-PICKUP: 2 COLETAS → 1 ENTREGA (BRASIL - SANDBOX) ===
function testar_lalamove_multi_pickup() {
    if (isset($_GET['test_lalamove_pickup']) && current_user_can('manage_options')) {
        
        echo '<div style="padding: 20px; background: #f0f0f1; margin: 20px; border-radius: 5px; font-family: Arial, sans-serif;">';
        echo '<h1>Multi-Pickup Test - 2 Vendedores → 1 Cliente (Fortaleza)</h1>';
        echo '<p><strong>Fluxo:</strong> Coletar em 2 lojas → Entregar no cliente final</p>';
        
        $api_key = 'pk_test_405b82b18afd1a3b961cb61c82e063cf';
        $secret = 'sk_test_8exxWZ9sH70NtO5Kt3yHsp9aDVY5OkhG0/J2hRDuxpXKkatG6OiwEZvA5Wr5TcGF';
        $market = 'BR';
        $base_url = 'https://rest.sandbox.lalamove.com';

        function generateUUID() {
            return sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
                mt_rand(0, 0xffff), mt_rand(0, 0xffff),
                mt_rand(0, 0xffff), mt_rand(0, 0x0fff) | 0x4000,
                mt_rand(0, 0x3fff) | 0x8000, mt_rand(0, 0xffff), 
                mt_rand(0, 0xffff), mt_rand(0, 0xffff)
            );
        }

        function makeLalamoveRequest($url, $headers, $body = null, $method = 'POST') {
            $args = ['headers' => $headers, 'timeout' => 30];
            if ($body && $method === 'POST') $args['body'] = $body;
            $response = $method === 'POST' ? wp_remote_post($url, $args) : wp_remote_get($url, $args);
            if (is_wp_error($response)) throw new Exception('Erro WP: ' . $response->get_error_message());
            return [
                'status' => wp_remote_retrieve_response_code($response),
                'body'   => wp_remote_retrieve_body($response)
            ];
        }

        try {
            // === 1. COTAÇÃO COM ITEM OBRIGATÓRIO ===
            echo '<h3>Criando Cotação (3 stops + item)</h3>';

            $quotationBody = [
                "data" => [
                    "serviceType" => "CAR",
                    "specialRequests" => [],
                    "language" => "pt_BR",
                    "stops" => [
                        ["coordinates" => ["lat" => "-3.7319", "lng" => "-38.5267"], "address" => "Av. Beira Mar, 3000 - Meireles"],
                        ["coordinates" => ["lat" => "-3.7172", "lng" => "-38.5433"], "address" => "Rua Silva Paulet, 1000 - Aldeota"],
                        ["coordinates" => ["lat" => "-3.7921", "lng" => "-38.4803"], "address" => "Av. Washington Soares, 85 - Edson Queiroz"]
                    ],
                    "isRouteOptimized" => false,
                    "item" => [
                        "weight" => "LESS_THAN_3_KG",
                        "categories" => ["FOOD_DELIVERY", "OTHERS"]
                    ]
                ]
            ];

            $bodyString = json_encode($quotationBody);
            $time = round(microtime(true) * 1000);
            $signature = hash_hmac('sha256', $time . "\r\nPOST\r\n/v3/quotations\r\n\r\n" . $bodyString, $secret);
            $token = $api_key . ':' . $time . ':' . $signature;

            $headers = [
                'Authorization' => 'hmac ' . $token,
                'Market' => $market,
                'Request-ID' => generateUUID(),
                'Content-Type' => 'application/json'
            ];

            $quotationResult = makeLalamoveRequest($base_url . '/v3/quotations', $headers, $bodyString, 'POST');
            if ($quotationResult['status'] !== 201) {
                throw new Exception("Erro na cotação: " . $quotationResult['status'] . " - " . $quotationResult['body']);
            }

            $quotationData = json_decode($quotationResult['body'], true);
            
            echo '<div style="background: #d4edda; padding: 10px; border-radius: 5px; margin: 10px 0;">';
            echo '<p><strong>COTAÇÃO OK!</strong> ID: ' . $quotationData['data']['quotationId'] . '</p>';
            echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' BRL</p>';
            echo '</div>';

            // === 2. ORDEM: 2 COLETAS (usando recipients) ===
            echo '<h3>Criando Ordem: 2 coletas → 1 entrega</h3>';

            $orderBody = [
                "data" => [
                    "quotationId" => $quotationData['data']['quotationId'],
                    "sender" => [
                        "stopId" => $quotationData['data']['stops'][0]['stopId'],
                        "name" => "Vendedor A",
                        "phone" => "+5585999999999"
                    ],
                    "recipients" => [
                        [
                            "stopId" => $quotationData['data']['stops'][1]['stopId'],
                            "name" => "Vendedor B",
                            "phone" => "+5585988888888",
                            "remarks" => "COLETAR PACOTE 2 - NÃO ENTREGAR"
                        ],
                        [
                            "stopId" => $quotationData['data']['stops'][2]['stopId'],
                            "name" => "Cliente Final",
                            "phone" => "+5585888888888",
                            "remarks" => "ENTREGAR 2 PACOTES (A + B)"
                        ]
                    ],
                    "isPODEnabled" => false
                ]
            ];

            $orderBodyString = json_encode($orderBody);
            $time2 = round(microtime(true) * 1000);
            $signature2 = hash_hmac('sha256', $time2 . "\r\nPOST\r\n/v3/orders\r\n\r\n" . $orderBodyString, $secret);
            $token2 = $api_key . ':' . $time2 . ':' . $signature2;

            $headers2 = [
                'Authorization' => 'hmac ' . $token2,
                'Market' => $market,
                'Request-ID' => generateUUID(),
                'Content-Type' => 'application/json'
            ];

            $orderResult = makeLalamoveRequest($base_url . '/v3/orders', $headers2, $orderBodyString, 'POST');

            if ($orderResult['status'] !== 201) {
                throw new Exception("Erro na ordem: " . $orderResult['status'] . " - " . $orderResult['body']);
            }

            $orderData = json_decode($orderResult['body'], true);
            
            echo '<div style="background: #d1ecf1; padding: 10px; border-radius: 5px; margin: 10px 0;">';
            echo '<p><strong>ORDEM CRIADA COM SUCESSO!</strong></p>';
            echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
            echo '<p><strong>Status:</strong> ' . $orderData['data']['status'] . '</p>';
            if (isset($orderData['data']['shareLink'])) {
                echo '<p><strong>Rastreio:</strong> <a href="' . $orderData['data']['shareLink'] . '" target="_blank">Acompanhar</a></p>';
            }
            echo '</div>';

            // === RESUMO ===
            echo '<div style="background: #fff3cd; padding: 15px; border-radius: 5px; margin: 20px 0;">';
            echo '<h3>RESUMO FINAL</h3>';
            echo '<p><strong>Quotation ID:</strong> ' . $quotationData['data']['quotationId'] . '</p>';
            echo '<p><strong>Order ID:</strong> ' . $orderData['data']['orderId'] . '</p>';
            echo '<p><strong>Preço:</strong> ' . $quotationData['data']['priceBreakdown']['total'] . ' BRL</p>';
            echo '<p><strong>Rota:</strong> 2 coletas → 1 entrega</p>';
            echo '</div>';

        } catch (Exception $e) {
            echo '<div style="background: #f8d7da; padding: 10px; border-radius: 5px;">';
            echo '<p><strong>ERRO:</strong> ' . htmlspecialchars($e->getMessage()) . '</p>';
            echo '</div>';
        }
        
        echo '<hr>';
        echo '<p><strong>Testar novamente:</strong> <a href="' . admin_url('?test_lalamove_pickup=1') . '">Clique aqui</a></p>';
        echo '</div>';
        die();
    }
}
add_action('init', 'testar_lalamove_multi_pickup');
