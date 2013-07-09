<?php if ( ! defined('BASEPATH')) exit('No direct script access allowed');

class Gamevongquay extends CI_Controller {
  
	function __construct()
	{
		session_start();
		parent::__construct();
		$this->view['title'] = 'Game vòng quay';
	}
	
	public function index()
	{
		if(!isset($this->session->userdata['Id_account']))
			header("Location: " . base_url() . "Thanhvien/Dangnhap/" . encodeReUrl("Gamevongquay"));
		
		$data = array();
		$flashParams = array(
				"AccountId" => $this->session->userdata['Id_account'],
				"VKim"		=> $this->session->userdata['bonusBal'],
				"SessionId"	=> session_id(),
				"api_addr"	=> $_SERVER["HTTP_HOST"]
		);
		$data['flashVars'] = http_build_query($flashParams);
		
		$this->view['content'] = $this->load->view('gamevongquay', array('data' => $data), true);
		$this->load->view('layout', array('data' => $this->view));
	}
	
	public function init()
	{
		if(isset($this->session->userdata['Id_account']))
		{
			$return = array();
			if(isset($_POST['cmd']) && $_POST['cmd'] == "init")
			{
				if(isset($this->session->userdata['Id_account']) && (isset($_POST['sessionid']) && $_POST['sessionid'] == session_id()))
				{
					$return['status']		= 1;
					$return['username']		= $this->session->userdata['Account_Name'];
					$return['totalpoint'] 	= $this->session->userdata['bonusBal'];
					$return['sessionid'] 	= session_id();
					echo json_encode($return);
					exit;
				}
			}
			$return['status'] = 0;
			echo json_encode($return);
		}
		exit;
	}
	
	public function play()
	{
		$return = array();
		$return['status'] = -2;
		if(isset($this->session->userdata['Id_account']))
		{
			$return['status'] = 0;
			if(isset($_POST['cmd']) && $_POST['cmd'] == "startturn")
			{
				if(isset($this->session->userdata['Id_account']) && (isset($_POST['sessionid']) && $_POST['sessionid'] == session_id()))
				{
					$this->load->model("Mcharge");
					$this->load->library('megacard_services');
					
					/*
					 * luanbv[2013-06-26] 
					* Kiem tra so du session
					*
					*/
					if($this->session->userdata['bonusBal'] > BET_POINT_PER_PLAY){
					
						// tru diem tren Epurse
						$result = $this->megacard_services->Payment(BET_POINT_PER_PLAY, $this->session->userdata);
						if($result->lmsRespCode == '00')
						{
						
							// tru sesssion
							$this->session->set_userdata('bonusBal', $this->session->userdata['bonusBal'] - BET_POINT_PER_PLAY);
						
							// log thay doi coupon
							$update_data = array(
								'Id_account'	=>	$this->session->userdata['Id_account'],
								'Created_Date'	=>	date("Y-m-d H:i:s"),
								'Type'			=>	1, // 0: cong diem; 1: tru diem
								'Comment'		=>	"Ch&#417;i Game v&#242;ng quay",
								'Amount'		=>	BET_POINT_PER_PLAY,
							);
							$this->Mcharge->addLog($update_data);
						
							// kiem tra lich su trung thuong + quay thuong
							$startPackage = $this->Mcharge->getStartPackage(BET_PACKAGE);
							$currentPst = ($startPackage['total_row'] == 0) ? 1 : $startPackage['total_row'];
							$configs = getConfigs();
							if($currentPst < $configs['bat_dau_chu_ky'])
								$turnresult = 0;
							else
							{
								if($currentPst > $configs['ket_thuc_chu_ky'])
								{
									$this->Mcharge->updateVongquayConfig($currentPst, $currentPst + BET_PACKAGE);
									$this->cache->memcached->delete(MEMCACHE_LIST_CONFIGS);
								}
								$memkey = MEMCACHE_CHU_KY_GAME_VONG_QUAY . $configs['bat_dau_chu_ky'];
								$plays = $this->cache->memcached->get($memkey);
								if(!$plays)
								{
									$plays  = array();
									$temp  = array();
									$endPackage = $configs['ket_thuc_chu_ky'];
									for($i = $startPackage['total_row']; $i <= $endPackage; ++$i)
									{
										$temp[$i] = 0;
										$plays[$i] = 0;
									}
									
									$rewards = array(
											REWARD_1_STATUS => 0,
											REWARD_2_STATUS => 0,
											REWARD_3_STATUS => 0,
											REWARD_4_STATUS => 0,
											REWARD_5_STATUS => 0,
											REWARD_6_STATUS => 0,
									);
									$reward_history = $this->Mcharge->getPlayHistory($configs['bat_dau_chu_ky']);
									if(!empty($reward_history))
									{
										foreach($reward_history as $item)
											++$rewards[$item['Turnresult']];
									}
									// tao danh sach luot quay
									foreach ($rewards as $key => $value)
									{
										switch ($key){
											case REWARD_1_STATUS:
												$max_reward = REWARD_1_MAX_TIME;
												break;
											case REWARD_2_STATUS:
												$max_reward = REWARD_2_MAX_TIME;
												break;
											case REWARD_3_STATUS:
												$max_reward = REWARD_3_MAX_TIME;
												break;
											case REWARD_4_STATUS:
												$max_reward = REWARD_4_MAX_TIME;
												break;
											case REWARD_5_STATUS:
												$max_reward = REWARD_5_MAX_TIME;
												break;
											case REWARD_6_STATUS:
												$max_reward = REWARD_6_MAX_TIME;
												break;
										}
										$num_reward = $max_reward - $value;
										if($num_reward > 0)
										{
											$rand_keys = array_rand($temp, $num_reward);
											foreach ($rand_keys as $rand)
											{
												$temp[$i] = 0;
												unset($temp['$rand']);
												$plays[$rand] = $key;
											}
										}
									}
									$this->cache->memcached->save($memkey, $plays, MEMCACHE_CHU_KY_GAME_VONG_QUAY_TIME);
								}
								$turnresult = $plays[$startPackage['total_row']];
								
								$id_game = $this->Mcharge->addVongquayLog(array('SessionID' => session_id(), 'Id_Account' => $this->session->userdata['Id_account']));
								$vongquay_data = array(
										'SessionID' 	=> session_id(),
										'Id_Account' 	=> $this->session->userdata['Id_account'],
										'Status' 		=> 1,
										'Turnresult' 	=> $turnresult,
								);
								
								// xu ly trung thuong
								if($turnresult > 0)
								{
									switch ($turnresult){
										case REWARD_1_STATUS:
										case REWARD_2_STATUS:
										case REWARD_3_STATUS:
										case REWARD_4_STATUS:
											if($turnresult == REWARD_1_STATUS ||$turnresult == REWARD_2_STATUS)
											{
												$reward = ($turnresult == REWARD_1_STATUS) ? REWARD_1 : REWARD_2;
												$result = ($turnresult == REWARD_1_STATUS) ? REWARD_1_TEXT : REWARD_2_TEXT;
												$cardSerie = getCardSeries($reward);
												$email_subject = EMAIL_TRUNG_THUONG_GAME_VONG_QUAY_SUBJECT;
												$email_content = sprintf(EMAIL_TRUNG_THUONG_GAME_VONG_QUAY_CONTENT, base_url(), $this->session->userdata('Account_Name'), $reward, $cardSerie['Pincode'], $cardSerie['Serial'], base_url());
											}
											else
											{
												$reward = ($turnresult == REWARD_3_STATUS) ? REWARD_3 : REWARD_4;
												$result = ($turnresult == REWARD_3_STATUS) ? REWARD_3_TEXT : REWARD_4_TEXT;
												$cardSerie = getCouponSeries($reward, 0);
												$this->updateCouponHistory($cardSerie, $id_game);
												$email_subject = EMAIL_TRUNG_THUONG_GAME_VONG_QUAY_SUBJECT;
												$email_content = sprintf(EMAIL_TRUNG_THUONG_GAME_VONG_QUAY_CONTENT_2, base_url(), $this->session->userdata('Account_Name'), $reward/1000 . " &#273;i&#7875;m", $cardSerie['Pincode'], base_url());
											}
											
											$this->email_config = Array(
													'protocol'  => 'smtp',
													'smtp_host' => SMTP_HOST,
													'smtp_port' => SMTP_PORT,
													'smtp_user' => SMTP_USER,
													'smtp_pass' => SMTP_PASSWORD,
													'mailtype'  => SMTP_TYPE,
													'starttls'  => SMTP_STARTTLS,
													'newline'   => "\r\n"
											);
											
	// 										$this->load->library('email', $this->email_config);
	// 										$this->email->from(SMTP_USER, SITE_NAME);
	// 										$this->email->to($this->session->userdata('Account_Email'));
	// 										$this->email->subject($email_subject);
	// 										$this->email->message($email_content);
	// 										$this->email->send();
											$client = new SoapClient(WS_SENDMAIL, array('encoding'=>'UTF-8'));
											try
											{
												$result = $client->sendEmail(SMTP_USER,SMTP_PASSWORD,$email_subject,$email_content,$this->session->userdata('Account_Email'),0);
											}
											catch (Exception $ex)
											{
												$vongquay_data['Account_Email'] = $this->session->userdata('Account_Email');
												$vongquay_data['Email_Content'] = $email_content;
												break;
													
											}	
											$vongquay_data['Account_Email'] = $this->session->userdata('Account_Email');
											$vongquay_data['Email_Content'] = $email_content;
											break;
											
											
										case REWARD_5_STATUS:
										case REWARD_6_STATUS:
											$reward = ($turnresult == REWARD_5_STATUS) ? REWARD_5 : REWARD_6;
											$result = ($turnresult == REWARD_1_STATUS) ? REWARD_5_TEXT : REWARD_6_TEXT;
											
											$this->load->library("saobacvietapi");
											$this->load->library("encript");
										
											$userInfo = $this->saobacvietapi->SetUserInfo($this->session->userdata['Account_Name'],$this->session->userdata['Id_account'].'@megacard.vn','mgc_'.$this->session->userdata['Id_account'],$this->session->userdata['session_id'],'MGC');
						
											$time = gettimeofday();
											$stan = $time['sec'];
											$epay_mess = $userInfo->AccountId . $stan;
											$encrypt_str = $this->encript->encryptRSA($epay_mess, VMG_RSA_PUBLIC_KEY);
											
											$ch = curl_init();
											curl_setopt($ch, CURLOPT_URL, WS_VMG_DOI_XU);
											curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
											curl_setopt($ch, CURLOPT_POST, true);
										
											$data1 = array(
												'uid'	=> $userInfo->AccountId,
												'xu'	=> $reward,
												'txn_id'=> $stan,
												'sign'	=> $encrypt_str,
											);
											
											curl_setopt($ch, CURLOPT_POSTFIELDS, $data1);
											$output = curl_exec($ch);
											$output = json_decode($output, true);
											
											$insert = array();
											$insert['uid']			= $data1['uid'];
											$insert['xu']			= $data1['xu'];
											$insert['txn_id']		= $data1['txn_id'];
											$insert['Epay_sign']	= $data1['sign'];
											$insert['Id_Account']	= $this->session->userdata['Id_account'];
											$insert['Type']			= 2;
											$insert['ForeignID']	= $id_game;
										
											if(isset($output['status']))
											{
												$insert['status'] = $output['status'];
												if(isset($output['balance']))
													$insert['balance'] = $output['balance'];
												if(isset($output['balance']))
													$insert['vmg_sign'] = $output['sign'];
											}
											$this->Mcharge->addXuLog($insert);
											curl_close($ch);
											break;
									}
								}
								
								// ghi log
								$this->Mcharge->updateVongquayLog($vongquay_data, $id_game);
								
								$trans = array(
										'Id_Products'	 => 0,
										'Id_Account'	 => $this->session->userdata['Id_account'],
										'Amount'		 => BET_POINT_PER_PLAY,
										'Id_Promotion'	 => 0,
										'Promotion_Value'=> 0,
										'ID_Game' 		 => $id_game,
										'Account_Name'	 => $this->session->userdata['Account_Name'],
										'Comment'		 => ($turnresult == 0) ? "Kh&#244;ng tr&#250;ng" : $result,
								);
								$this->Mcharge->addTransLog($trans);
								
								echo $turnresult;
								exit;
							}
							
						}
						$return['status'] = -1;
					}else{
						$return['status'] = 2; //Khong du diem de quay tay
					}
				}
			}
		}
		echo $return['status'];
		exit;
	}
	
	private function updateCouponHistory($coupon, $id_game)
	{
		$this->load->model("Mcharge");
		if(!$this->Mcharge->UpdateCoupon(array('Status' => 1, 'Id_VongquayLog' => $id_game, 'Type' => 1), $coupon['Id_Coupon']))
		{
			$this->Mcharge->UpdateCoupon(array('Status' => 1, 'Id_VongquayLog' => $id_game, 'Type' => 1), $coupon['Id_Coupon']);
		}
	}
	
	private function getCouponSeries($price, $Id_VongquayLog)
	{
		$price = $price * 1000;
		$this->load->model("Mcharge");
		$key = MEMCACHE_LIST_COUPON_BY_PRICE . $price;
		$coupon = false;
		$i = 0;
		do {
			++$i;
			$data = $this->cache->memcached->get($key);
			if(!$data || empty($data))
			{
				$data = $this->Mcharge->getListCoupon($price, NUM_COUPON_PER_TIME);
			}
			$j = 0;
			do {
				++$j;
				$coupon = array_shift($data);
				if((strtotime($coupon['Expire_Date']) - time()) < COUPON_EXPIRE_TIME)
					$coupon = false;
				if($j == 3)
					return false;
			} while ($coupon == false);
			if($this->Mcharge->UpdateCoupon(array('Status' => 1, 'Id_VongquayLog' => $Id_VongquayLog, 'Type' => 1), $coupon['Id_Coupon']))
				$this->cache->memcached->save($key, $data, MEMCACHE_LIST_COUPON_BY_PRICE_TIME);
			else
			{
				if($this->Mcharge->UpdateCoupon(array('Status' => 1, 'Id_VongquayLog' => $Id_VongquayLog, 'Type' => 1), $coupon['Id_Coupon']))
					$this->cache->memcached->save($key, $data, MEMCACHE_LIST_COUPON_BY_PRICE_TIME);
			}
			if($i == 3)
				return false;
		}while ($coupon == false);
	
		$this->load->library('megacard_services');
		$pincode = $this->megacard_services->decryptCardSeries($coupon['Pincode'], CARD_DECRYPT_KEY);
		if(strlen($pincode) < 6)
			$pincode = str_pad($pincode, 6, "0", STR_PAD_LEFT);
		return array('Price_Coupon' => $coupon['Price_Coupon'], 'Pincode' => $pincode);
	}
}
