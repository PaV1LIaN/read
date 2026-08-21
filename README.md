const form = document.querySelector('#userForm');
const nameInput = document.querySelector('#name');
const ageInput = document.querySelector('#age');

form.addEventListener('submit', async(event) => {
    event.preventDefault();

    const name = nameInput.value.trim();
    const age = Number(ageInput.value);

    const user = {
        name,
        age
    };
    
    console.log(user);
});
