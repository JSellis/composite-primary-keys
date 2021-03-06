[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/maksimru/composite-primary-keys/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/maksimru/composite-primary-keys/?branch=master)
[![codecov](https://codecov.io/gh/maksimru/composite-primary-keys/branch/master/graph/badge.svg)](https://codecov.io/gh/maksimru/composite-primary-keys)
[![StyleCI](https://github.styleci.io/repos/163864737/shield?branch=master)](https://github.styleci.io/repos/163864737)
[![CircleCI](https://circleci.com/gh/maksimru/composite-primary-keys.svg?style=svg)](https://circleci.com/gh/maksimru/composite-primary-keys)

## About

Library extends Laravel's Eloquent ORM with pretty full support of composite keys

## Usage

Laravel 5.5

```shell script
composer require maksimru/composite-primary-keys ~0.14
```

Laravel 5.6+

```shell script
composer require maksimru/composite-primary-keys ~1.0
```

Simply add \MaksimM\CompositePrimaryKeys\Http\Traits\HasCompositePrimaryKey trait into required models

## Features Support

  
- increment and decrement
- update and save query
- binary columns
    
    ```php  
    class BinaryKeyUser extends Model
    {
        use \MaksimM\CompositePrimaryKeys\Http\Traits\HasCompositePrimaryKey;
    
        protected $binaryColumns = [
            'user_id'
        ];
    
        protected $primaryKey = 'user_id';
    }
    ```
  
  With $hexBinaryColumns = false or omitted, $binaryKeyUser->user_id will return binary value. BinaryKeyUser::find('BINARY_VALUE') and BinaryKeyUser::create(['id' => 'BINARY_VALUE']) should be used in this case
  
  Optional ability to automatically encode binary values to their hex representation:
  
    ```php  
    class BinaryKeyUser extends Model
    {
      use \MaksimM\CompositePrimaryKeys\Http\Traits\HasCompositePrimaryKey;
    
      protected $binaryColumns = [
          'user_id'
      ];
    
      protected $primaryKey = 'user_id';
    
      protected $hexBinaryColumns = true;
    }
    ```
  
  With $hexBinaryColumns = true, $binaryKeyUser->user_id will return hex value. BinaryKeyUser::find('HEX_VALUE') and BinaryKeyUser::create(['id' => 'HEX_VALUE']) should be used in this case
  
  *JSON output will contain hex values in both cases*
  
- model serialization in queues (with Illuminate\Queue\SerializesModels trait)

    Job:
    
    ```php
    class TestJob implements ShouldQueue
    {
        use Queueable, SerializesModels;
    
        private $model;
    
        /**
         * Create a new job instance.
         */
        public function __construct(TestUser $testUser)
        {
            $this->model = $testUser;
        }
    
        /**
         * Execute the job.
         */
        public function handle()
        {
            $this->model->update(['counter' => 3333]);
        }
    }
    ```
    
    Dispatch:
    
    ```php
    $model = TestUser::find([
        'user_id' => 1,
        'organization_id' => 100,
    ]);
    $this->dispatch(new TestJob($model));
    ```
    
- route implicit model binding support
  
    Model:
    
    ```php
    class TestBinaryUser extends Model
    {
        use \MaksimM\CompositePrimaryKeys\Http\Traits\HasCompositePrimaryKey;
        
        protected $table = 'binary_users';
        
        public $timestamps = false;
        
        protected $binaryColumns = [
          'user_id'
        ];
        
        protected $primaryKey = [
          'user_id',
          'organization_id',
        ];
    }
    ```
    
    routes.php:
    
    ```php
    $router->get('binary-users/{binaryUser}', function (BinaryUser $binaryUser) {
        return $binaryUser->toJson();
    })->middleware('bindings')
    ```
    
    request:
    
    ```http request
    GET /binary-users/D9798CDF31C02D86B8B81CC119D94836___100
    ```
    
    response:
    
    ```json
    {"user_id":"D9798CDF31C02D86B8B81CC119D94836","organization_id":"100","name":"Foo","user_id___organization_id":"D9798CDF31C02D86B8B81CC119D94836___100"}
    ```

- relations (only belongsTo relation is supported in this version)

  ```php
  class TestUser extends Model
  {
      use \MaksimM\CompositePrimaryKeys\Http\Traits\HasCompositePrimaryKey;
  
      protected $table = 'users';
  
      protected $primaryKey = [
          'user_id',
          'organization_id',
      ];
  
      public function referrer()
      {
          return $this->belongsTo(TestUser::class, [
              'referred_by_user_id',
              'referred_by_organization_id'
          ], [
              'user_id',
              'organization_id',
          ]);
      }
  }

  $referrer_user = $testUser->referrer()->first();
  ```
  
  will call
  
  ```sql
  select * from "users" where (("user_id" = ? and "organization_id" = ?)) limit 1
  ```
  
  with bindings [ $testUser->referred_by_user_id, $testUser->referred_by_organization_id ] 