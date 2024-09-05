# datatable

# controller 
  function opdListByDatatable(){
	    
		$session = $this->session->userdata();
		$startTime = microtime(true);
		$opd = $this->Appointment_model->opdListing($session, $_REQUEST);
		$endTime = microtime(true);

		$data = [];
		if( $opd ){
			foreach ($opd as $key => $value) {

				$view = site_url("admin/appointment/details/{$value->id}");
				$edit = site_url("admin/appointment/save/{$value->id}");
				$iaf = site_url("doctor/appointment/patient_form/{$value->id}")."?iaf=".encrypt($value->uhid);
				$printInvoice = site_url("admin/registration/appointment/{$value->id}");

				$action = "<a href='{$view}' target='_blank' class='action-icon' title='view'> <i class='zmdi zmdi-eye'></i></a>";

				if(date('d-m-Y') == date("d-m-Y", strtotime($value->created_at))){ 
					$action .= "<a href='{$edit}' target='_blank' class='action-icon' title='edit'> <i class='zmdi zmdi-edit'></i></a>";
				}

				$action .= "<a href='{$iaf}' target='_blank' class='action-icon' title='Patient form'> <i class='zmdi zmdi-download'></i></a>";

				if( $value->date_app == date('Y-m-d') ){ 
					$action .= "<a href='{$printInvoice}' target='_blank' class='action-icon' title='Print Invoice & Prescription Pad'> <i class='zmdi zmdi-file'></i></a>";
				}
				
				$action .= "<a href='javascript:;' title='Exchange' data-id='{$value->id}' data-url='admin/appointment/doctorExchange' class='action-icon customPopup' title='Patient form'><i class='fa fa-exchange' aria-hidden='true'></i></a>";
				
				$data[] = [
					($_REQUEST['start'] + 1 + $key),
					$value->branch_name,
					"<a href = '{$view}' target='_blank' title='view' >{$value->uhid}</a>",
					$value->token,
					$value->id,
					$value->patient_name,
					$value->doctor_name,
					date("d-m-Y", strtotime($value->date_app)),
					$value->time_slot,
					$value->gender,
					$value->age,
					$value->patient_type,
					$value->city,
					$value->state,
					$value->country,
					date("d-m-Y h:i A", strtotime($value->created_at)),
					$action

				];
			}
		}
		
		$totalRecord = $this->Appointment_model->opdCountRecord($session, $_REQUEST);
		


		$json_data = [
			"draw"            => intval( $_REQUEST['draw'] ),
			"recordsTotal"    => $totalRecord,
			"recordsFiltered" => $totalRecord,
			"data"            => $data,
			"timeTaken"       => $endTime - $startTime
		];
		echo json_encode($json_data);
	}

# model 

function opdGetListing($session, $request){
		
	$this->db->select('appointment.id, appointment.uhid, appointment.token, appointment.date_app, appointment.time_slot, appointment.created_at, registration.patient as patient_name ,registration.contact_number, registration.address,registration.city,registration.state,registration.country,registration.contact_number,registration.patient_type,registration.gender,registration.age,registration.blood_group, doctor.id as doctor_id, doctor.name as doctor_name,branch.name as branch_name,panel_patient_payment.file_status,panel_patient_payment.payment_status')
						->from('appointment')
						->join('registration', 'registration.id=appointment.patient_id', 'LEFT')
						->join('doctor', 'doctor.id=appointment.doctor_id', 'LEFT')
						->join('branch', 'branch.id=appointment.branch_id', 'LEFT')
						->join('panel_patient_payment', 'panel_patient_payment.patient_id = appointment.id', 'LEFT');

		if( $session['login_branch']  >  0){
			$this->db->where('appointment.branch_id', $session['login_branch']);
		}
		
		$this->db->where('appointment.status','active');


		if( !empty($request['columns'][2]['search']['value'] ) ){
			$this->db->where('appointment.uhid LIKE', "{$request['columns'][2]['search']['value']}%");
		}elseif(!empty($request['columns'][5]['search']['value'] )){
			$this->db->where('registration.patient LIKE', "%{$request['columns'][5]['search']['value']}%");
		}else{
			
			if( !empty($request['columns'][7]['search']['value'] ) ){
				$explodeDate = explode('__',$request['columns'][7]['search']['value']);
				$startDate = $explodeDate[0];
				$endDate = $explodeDate[1];
				$this->db->where('appointment.created_at >=', date('Y-m-d 00:00:00', strtotime($startDate)))
								->where('appointment.created_at <=', date('Y-m-d 23:59:59', strtotime($endDate)));
			}else{
				$this->db->where('appointment.created_at >=', date('Y-m-d 00:00:00', strtotime('-30 days')))
								->where('appointment.created_at <=', date('Y-m-d 23:59:59'));
			}
		}

		
		if( !empty($request['columns'][6]['search']['value'] ) ){
			$this->db->where('doctor.id', $request['columns'][6]['search']['value']);
		}

		if( !empty($request['columns'][1]['search']['value'] ) ){
			$this->db->where('appointment.branch_id', $request['columns'][1]['search']['value']);
		}

		if( !empty($request['search']['value']) ){
			$this->db->group_start()
						->where('appointment.uhid LIKE', "{$request['search']['value']}%")
						->where('registration.patient LIKE', "%{$request['search']['value']}%")
						->or_where('registration.contact_number LIKE', "{$request['search']['value']}%")
						->group_end();
		}

	}

	function opdCountRecord($session, $request){
		$this->opdGetListing($session, $request);
		return $this->db->count_all_results();
	}

	function opdListing($session, $request){
		$this->opdGetListing($session, $request);
		return $this->db->order_by('appointment.created_at', 'desc')
		->limit($request['length'], $request['start'])->get()->result();

	}

# jquery 

const baseUrl = $('meta[name=base_url]').attr('content');
const csrfToken = $('meta[name=csrf_test_name]').attr('content');
const csrfName = 'csrf_test_name';

if( $('#opdDatatable').length || $('#dischargeDatatable').length || $('#walletListDatatable').length ){

	var limitDays = 30;
	var start = moment().subtract(limitDays, 'days');
	var end = moment();
	
	$('input[name="dates"]').daterangepicker({
		autoUpdateInput: false,
		locale: {
			format: 'MMMM D, YYYY'
		},
		startDate: start,
        endDate: end,
        ranges: {
			'Today': [moment(), moment()],
			'Yesterday': [moment().subtract(1, 'days'), moment().subtract(1, 'days')],
			'Last 7 Days': [moment().subtract(6, 'days'), moment()],
			'Last 30 Days': [moment().subtract(29, 'days'), moment()],
			'This Month': [moment().startOf('month'), moment().endOf('month')],
			'Last Month': [moment().subtract(1, 'month').startOf('month'), moment().subtract(1, 'month').endOf('month')]
        }
	}, cb);
	
	cb(start, end);
	
	
    function cb(start, end) {
		$('input[name="dates"]').val(start.format('MMMM D, YYYY') + ' - ' + end.format('MMMM D, YYYY'));
    }

	function debounce(func, delay) {
		let timeoutId;
		return function() {
		  clearTimeout(timeoutId);
		  timeoutId = setTimeout(() => {
			func.apply(this, arguments);
		  }, delay);
		};
	  }
	
	$(document).ready(function(){
		$('input[name="dates"]').val(moment().subtract(limitDays, 'days').format('MMMM D, YYYY') + ' - ' + moment().format('MMMM D, YYYY'));
		$('.select2Branch').val('').trigger('change');
		$('.select2Doctor').val('').trigger('change');
	});

	var select2Branch = $('.select2Branch').select2();
	var select2Doctor = $('.select2Doctor').select2();

	
	if( $('#opdDatatable').length ){
		let opdDatatable = $('#opdDatatable').DataTable({
			"serverSide": true,
			processing: true,
			stateSave: true ,
			"ordering": false,
			order: [
				[1, 'desc'],
			],
			"ajax": {
				"url"  : `${baseUrl}admin/appointment/opdListByDatatable`,
				"type" : "GET"
			},
			"initComplete": function () {
				$('#skeleton').hide();
				$('#opdDatatable').show();

				this.api()
					.columns([2, 5, 6, 1])
					.every(function () {
						let column = this;
						let indexNumber = column.index();

						let onlySearchFiled = [2, 5];

						if( onlySearchFiled.includes(indexNumber) ){
							let div = document.createElement('div');
							div.className = "form-group has-search position-relative SerachDatatableFilter";
							div.style.width = "135px";
	
							let input = document.createElement('input');
							input.placeholder = 'Search...';
							input.className = 'form-control';
	
							let span = document.createElement('span');
							span.className = 'fa fa-times form-control-feedback clearField';
							span.setAttribute('columnindex', indexNumber);
							span.setAttribute('isdraw', 'no')
							div.appendChild(input);
							
							column.header().append(div);
							input.addEventListener('keyup', () => {
								let lengthValue = 0;
								if( indexNumber == 2 ){
									lengthValue = 7;
								}else if( indexNumber == 5 ){
									lengthValue = 5;
								}	
								if( input.value.length > 0  ){
									div.appendChild(span);
									span.style.display = 'block';
									span.setAttribute('checklength', lengthValue)
									span.setAttribute('inputlength', lengthValue)
								}
								if( input.value !== '' && input.value.length > lengthValue && column.search() !== input.value ){
									span.setAttribute('isdraw', 'yes')
									column.search(input.value).draw()
								}
							});
						}

						let dropDownSelect = [6];
						if( $('#loginBranchId').val() == 0 ){ 
							dropDownSelect.push(1);
						}

						if( dropDownSelect.includes(indexNumber) ){

							let div = document.createElement('div');
							div.style.width = "170px";
							let select = document.createElement('select');
							
							var selectClass = 'select2Doctor';
							var allOptionVal = $('#allDoctor').val();
							if( indexNumber == 6 ){
								selectClass = 'select2Doctor';
								allOptionVal = $('#allDoctor').val();
							}else if( indexNumber == 1 ){
								selectClass = 'select2Branch';
								allOptionVal = $('#allBranch').val();
							}
							select.className = selectClass;
							
							if( allOptionVal !== '' ){
								allOptionVal = JSON.parse(allOptionVal)
								
								let option = document.createElement("option");
								option.text = "..Select..";
								option.value = "";
								select.appendChild(option);
								for (let data of allOptionVal) {
									let option = document.createElement("option");
									option.text = data.name;
									option.value = data.id;
									select.appendChild(option);
								}
								setTimeout(function(){
									$(`.${selectClass}`).select2()
								}, 50);
							}
							
							div.append(select);
							column.header().append(div);

							$(`.${selectClass}`).on('select2:select', function (e) {
								var data = e.params.data.id;
								column.search(data).draw();
							});
							
						}

	
					});

			}
		});


		$(document).on('click', '.clearField', function(){
			let columnindex = $(this).attr('columnindex');
			let isdraw = $(this).attr('isdraw');
			$(this).siblings('input').val('');
			if( isdraw == 'yes' ){
				opdDatatable.columns(columnindex).search('').draw();
			}
			$(this).hide();
		})
	
		if( $('#loginBranchId').val() != 0 ){
			opdDatatable.on('draw.dt', function () {
				opdDatatable.column(1).visible(false);
			});
		}else{
			opdDatatable.on('draw.dt', function () {
				opdDatatable.column(1).visible(true);
			});
		}
	
	
		$('input[name="dates"]').on('apply.daterangepicker', function(ev, picker) {
		  $(this).val(picker.startDate.format('MMMM D, YYYY') + ' - ' + picker.endDate.format('MMMM D, YYYY'));
		  opdDatatable.columns(7).search(picker.startDate.format('MMMM D, YYYY') + '__' + picker.endDate.format('MMMM D, YYYY')).draw();
		});

		$('.select2Branch').on('select2:select', function (e) {
			var data = e.params.data.id;
			opdDatatable.columns(1).search(data).draw();
		});

		$(document).on('click', '#resetOpdDatatable', function(){
			localStorage.clear();
			$('input[name="dates"]').val(moment().subtract(limitDays, 'days').format('MMMM D, YYYY') + ' - ' + moment().format('MMMM D, YYYY'));
			$('.select2Branch').val('').trigger('change');
			$('.select2Doctor').val('').trigger('change');
			$('.SerachDatatableFilter').children('input').val('');
			$('.SerachDatatableFilter .clearField').hide();
			opdDatatable.columns([1, 2, 6, 5, 7]).search('').draw();
		})
		
		$(window).on('load', function() {
		   localStorage.clear();
		   $('#opdDatatable_processing').hide();
		   let checkValueExist = [1, 2, 6, 5, 7];
		   let drawColumn = [];
		   for( let value of checkValueExist ){
		       if( opdDatatable.columns(value).search()[0] != '' ){
		            drawColumn.push(value)
		       }
		   }
		   if( drawColumn.length > 0 ){
		       opdDatatable.columns(drawColumn).search('').draw();
		   }
		});

		

	}


