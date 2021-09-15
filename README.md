# paypal-subscription



this method is for only one plan if you have only one subscription plan if you have more than one subscription plan like 3 plan (basic,medium,pro)
then you have to make save your paypla plan_id in a database table and fetch plan_id from database table and use in function where i am using plan id

Step1:
create model of subscriptions with migration 

Step2:
Past this code in subscription_migration

            $table->string('subscription_id')->nullable();
            $table->string('amount')->nullable();
            $table->string('create_date')->nullable();
            $table->string('currency')->nullable();
            $table->string('status')->nullable();
            $table->string('customer_id')->nullable();
            $table->string('start_date')->nullable();
            $table->string('end_date')->nullable();
            $table->string('method')->nullable();
            $table->string('paypal_plan_id')->nullable();
            
  Step3:
  run php artisan migrate
  
  Step4:
  create plan for subscription in payple
  
  Step5:
  past paypale key , secret and plan id in .env file
  
    PAYPAL_CLIENT_ID=AYE67-8J9jwYjARo0A3hUcbh4Ru_gkSp_abcabcabcabcabcabc---this is for demo
    PAYPAL_CLIENT_SECRET=EMv-mwXghDUO4rYdHjDUUh4Ru_gkSp_abcabcabcabcabcabc---this is for demo
    PAYPAL_PLAN_ID=P-9Y0017_abcabcabcabcabcabc---this is for demo
    
    step 6:
    this is a trait and past it in app/trait
    
    
    <?php

namespace App\Traits;
use GuzzleHttp\Exception\GuzzleException;
use GuzzleHttp\Exception\ClientException;
use GuzzleHttp\Pool;
use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Request;
use function GuzzleHttp\Promise\settle;

trait PayPalPlansApi{
    private $key;
    protected $host;
    protected $header;

    public function __construct()
    {
    //here you have to past yor paypal account key id
        $this->key = 'QVlFNjctOEo5andZakFSbzBBM2hVY2JoNFJ1X2drU3BfbDB2cVNnRFJtVmgteWxIbnJyNWJQR3pDTnB2SU9PX0NQOU1vRXBxM2U3TzEzaVo6RU12LW13WGdoRFVPNHJZZEhqRFVVQldIMlFFMUpkcDh5VEZxM2RWa0pMMzZFUzM2Vlo0cUZmQmo4SVA3VkhBOTFJN2ZhdlpaRmRoUjg3R08=';
        //this is ok
        $this->host = 'https://api.sandbox.paypal.com/v1/billing/plans?product_id=LiveLearning231&page_size=3&page=1&total_required=true';

        $this->header = [
            'Authorization' => 'Basic '.$this->key,
        ];
    }


    public function paypalSubscription($plan_id,$user_id)
    {

        $url = "https://api.sandbox.paypal.com/v1/billing/subscriptions";

        $client = new Client();
        $response = $client->request('POST', $url, [
            'headers' => [
                'Accept' => 'application/json',
                'Content-Type' => 'application/json',
                'Authorization' => 'Basic '.$this->key
            ],
                'json' => [
                    "plan_id" => $plan_id,

                    'application_context' => [
                        "return_url" => url("paymentsuccess/".$user_id),
                        "cancel_url" => url("paymenterror")
                    ]

                ]

            ]
        );


        $response = $response->getBody()->getContents();

        $response = json_decode($response);
        $link = $response->links['0']->href;

        return $link;


        }



 public function payment_success(Request $request){

        $uri = 'https://api.sandbox.paypal.com/v1/billing/subscriptions/'.$request->subscription_id;


        $client = new Client();
        $response = $client->get($uri, [
            'headers' => $this->header,
        ]);

        $response = json_decode($response->getBody(), true);

        $currency_code = $response['billing_info']['outstanding_balance']['currency_code'];
        $value = $response['billing_info']['outstanding_balance']['value'];
        $payer_id = $response['subscriber']['payer_id'];
        $next_billing_time = $response['billing_info']['next_billing_time'];
        $status_update_time = $response['status_update_time'];
        $start_time = $response['start_time'];
        $plan_id = $response['plan_id'];

        //dd($response);
        $subscription = Subscription::where('subscription_id', $request->subscription_id)->first();

        if( $response['status'] == "ACTIVE" && $subscription == null){

            $subscription = Subscription::create([
                'subscription_id' => $request->subscription_id,
                'user_id' => Auth()->user()->id,
                'name'=>'plan name',
                'amount' => $value,
                'create_date' => $start_time,
                'currency' => $currency_code,
                'status' => $response['status'],
                'customer_id' => $payer_id,
                'start_date' => $status_update_time,
                'end_date' => $next_billing_time,
                'method' => 'paypal',
                'paypal_plan_id' => $plan_id,

            ]);

        }
        return redirect('/');


    }
    public function payment_error(){
        return 'payment not done! Error';
    }



}





step 7:
use these method where you want in controller ( when click on pay with paypal and request come in controller method call this function)

    public function proceedPaymentPaypal($request){

        $user = Auth::user();
        $user_id = $user->id;
        $plan_id    = env("PAYPAL_PLAN_ID");
        $email      = $user->email;

        //$p_id = 'P-7JW45276T0402174UL52KUCI';

        $link = $this->paypalSubscription($plan_id, $user_id);
        return $link;

    }
    
    
    step:8
    route when you get subscription what is the route you want to hit 
    
    Route::get('/paymentsuccess/{id}',[ProfileController::class,'payment_success']);
Route::get('/paymenterror',[ProfileController::class,'payment_error']);
    
